# Fix instructions: employee ID collisions + audit log

Context: see project doc `claude/employee_upsert_overwrite_incident_2026-09-17.md` for the
full incident writeup (3 employees silently overwritten: Jacob Crouch, Al Clark, Curtis Moyer).
This spec covers the two outstanding code fixes it calls for.

## Part 1: Fix the ID-collision / silent-overwrite bug

### Root cause (confirmed in code)

`brew_culture_ap.html` generates every record ID client-side from a **localStorage counter**,
completely disconnected from what's actually in the database:

```javascript
const load=k=>{try{return JSON.parse(localStorage.getItem(k)||'null')}catch{return null}};
const save=(k,v)=>localStorage.setItem(k,JSON.stringify(v));
function nextSeq(p){const s=load('bc_seq')||{};s[p]=(s[p]||0)+1;save('bc_seq',s);return`${p}-${String(s[p]).padStart(4,'0')}`}
```

And the save pattern used everywhere (shown here for vendors, but employees follows the same
shape) does an **upsert**, not a true insert, for new records:

```javascript
const vendor = editId ? {...DB.vendors.find(v=>v.id===editId),...data} : {id:nextSeq('VND'),...data};
const ok = await saveVendorToSB(vendor); // -> sbUpsert -> .upsert(arr, {onConflict:'id'})
```

Combine the two and you get exactly what happened three times: `nextSeq('EMP')` hands out
`EMP-0001`, `EMP-0002`, `EMP-0003`... starting from whatever count happens to be sitting in
*that browser's* localStorage -- which was never incremented for the 14 employees that were
migrated straight into Supabase (bypassing the UI). So the very next "Add Employee" submitted
from a browser with a stale/zeroed counter reuses an ID that already belongs to a real employee,
and `.upsert(..., {onConflict:'id'})` silently overwrites that row instead of failing.

This same `nextSeq()` + upsert pattern is also used for vendors (`VND`), invoices (`INV`),
payments (`PMT`), and GL entries (`JE`/`GL`) -- lower risk today per the existing reference doc
("only one person enters invoices and payments" from one device), but it's the same latent bug.
Worth fixing there eventually too; not required for this pass.

### Required fix

**1. Stop minting IDs from localStorage. Get the next ID from the database instead.**

Add a Postgres sequence and RPC function so the ID authority lives in the DB, not the browser:

```sql
CREATE SEQUENCE IF NOT EXISTS employees_id_seq START WITH 18; -- next free after EMP-0017

CREATE OR REPLACE FUNCTION public.next_employee_id()
RETURNS TEXT LANGUAGE sql AS $$
  SELECT 'EMP-' || lpad(nextval('employees_id_seq')::text, 4, '0');
$$;
```

In the app, replace `nextSeq('EMP')` for the employee-add path with:

```javascript
async function nextEmployeeId(){
  const {data, error} = await _sb.rpc('next_employee_id');
  if(error){ console.error('nextEmployeeId error', error); throw error; }
  return data;
}
```

(Do the same for `employee_pay_rates` -- add an `employee_pay_rates_id_seq` /
`next_pay_rate_id()` pair, since that table was upserted-and-overwritten in every one of the
three incidents too.)

**2. Use a real insert for brand-new employees; only upsert when editing a known existing ID.**

Find the employee save function (same shape as `saveVendor` above -- search for where the
Employee modal's Save button calls out). Change it from:

```javascript
const emp = editId ? {...existing, ...data} : {id: nextSeq('EMP'), ...data};
await saveEmployeeToSB(emp); // upsert either way
```

to:

```javascript
if (editId) {
  const emp = {...existing, ...data};
  await sbUpsert('employees', emp); // edit = upsert is correct here
} else {
  const emp = {id: await nextEmployeeId(), ...data};
  const ok = await sbInsert('employees', emp); // TRUE insert -- fails loudly on PK collision
  if (!ok) { showError('Could not create employee -- ID collision, please retry.'); return; }
}
```

`sbInsert` already exists in the file (`.insert()`, used elsewhere) -- just route the
new-employee path through it instead of through `saveEmployeeToSB`'s upsert. Apply the same
insert-vs-upsert split to whatever function writes `employee_pay_rates` rows.

This doesn't just fix the root cause (DB-authoritative IDs) -- it also means if a collision
*ever* happens again for any other reason, Postgres throws a primary-key-violation error the
user sees immediately, instead of silently destroying another employee's record.

## Part 2: Add the audit log

This has been a known gap since `admin_module_spec.md` flagged it as a to-do; this incident is
the concrete case for building it now. Use a database trigger, not app-level logging calls --
a trigger can't be forgotten in some code path and also catches direct edits made from the
Supabase dashboard/SQL editor (which is how this incident had to be fixed).

```sql
CREATE TABLE IF NOT EXISTS public.audit_log (
  id BIGSERIAL PRIMARY KEY,
  table_name TEXT NOT NULL,
  row_id TEXT NOT NULL,
  action TEXT NOT NULL,              -- 'INSERT' | 'UPDATE' | 'DELETE'
  changed_by TEXT,                   -- from auth.jwt(), falls back to 'unknown'
  old_data JSONB,
  new_data JSONB,
  changed_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

ALTER TABLE public.audit_log ENABLE ROW LEVEL SECURITY;
-- Only readable by admin role; nothing should ever INSERT into it directly except the trigger.
CREATE POLICY audit_log_admin_read ON public.audit_log
  FOR SELECT USING ( (auth.jwt() -> 'user_metadata' ->> 'role') = 'admin' );

CREATE OR REPLACE FUNCTION public.log_audit_event()
RETURNS TRIGGER LANGUAGE plpgsql SECURITY DEFINER AS $$
DECLARE
  v_row_id TEXT;
  v_user TEXT;
BEGIN
  v_row_id := COALESCE(NEW.id, OLD.id);
  BEGIN
    v_user := COALESCE(auth.jwt() ->> 'email', 'unknown');
  EXCEPTION WHEN OTHERS THEN
    v_user := 'unknown';
  END;

  IF TG_OP = 'DELETE' THEN
    INSERT INTO public.audit_log(table_name, row_id, action, changed_by, old_data, new_data)
    VALUES (TG_TABLE_NAME, v_row_id, TG_OP, v_user, to_jsonb(OLD), NULL);
    RETURN OLD;
  ELSIF TG_OP = 'UPDATE' THEN
    INSERT INTO public.audit_log(table_name, row_id, action, changed_by, old_data, new_data)
    VALUES (TG_TABLE_NAME, v_row_id, TG_OP, v_user, to_jsonb(OLD), to_jsonb(NEW));
    RETURN NEW;
  ELSE
    INSERT INTO public.audit_log(table_name, row_id, action, changed_by, old_data, new_data)
    VALUES (TG_TABLE_NAME, v_row_id, TG_OP, v_user, NULL, to_jsonb(NEW));
    RETURN NEW;
  END IF;
END;
$$;

DROP TRIGGER IF EXISTS trg_audit_employees ON public.employees;
CREATE TRIGGER trg_audit_employees
AFTER INSERT OR UPDATE OR DELETE ON public.employees
FOR EACH ROW EXECUTE FUNCTION public.log_audit_event();

DROP TRIGGER IF EXISTS trg_audit_employee_pay_rates ON public.employee_pay_rates;
CREATE TRIGGER trg_audit_employee_pay_rates
AFTER INSERT OR UPDATE OR DELETE ON public.employee_pay_rates
FOR EACH ROW EXECUTE FUNCTION public.log_audit_event();
```

Start with just these two tables (the ones that broke). Extending the same trigger to
`shift_log`, `payroll_run_employees`, `payroll_run_line_items`, and eventually every mutable
table in the system is straightforward from here -- same function, one `CREATE TRIGGER` per
table -- and matches the "audit log" line item already sitting in `admin_module_spec.md`'s
build order.

Optional but recommended: a simple read-only "History" panel on the employee edit modal that
queries `audit_log WHERE table_name = 'employees' AND row_id = :id ORDER BY changed_at DESC` --
this turns "did someone else edit this today?" into a 2-second look instead of the multi-hour
log/data forensics this incident required.

## Verification after implementing

1. Add a test employee twice in a row from a browser with a cleared localStorage (simulating
   the stale-counter scenario) -- confirm the second add either gets a fresh ID or throws a
   visible error, and does **not** overwrite the first.
2. Confirm `audit_log` picks up a row for both the insert and a subsequent edit of that test
   employee, with correct old/new JSON.
3. Delete the test employee and its audit rows once confirmed.
4. Re-run the sweep queries from the incident doc's "Full-roster sweep" section against the
   live roster periodically until this ships, since the underlying bug is still live in
   production until Part 1 is deployed.
