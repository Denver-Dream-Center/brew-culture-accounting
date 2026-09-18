# Fix addendum: payroll_runs / payroll_run_employees / payroll_run_line_items ID collisions

Context: see project docs `claude/employee_upsert_overwrite_incident_2026-09-17.md` and
`claude/employee_id_collision_and_audit_log_fix_spec.md` for the original incident (3 employees
silently overwritten by the same bug on the `employees` and `employee_pay_rates` tables).

This addendum extends the identical fix to the three payroll tables, which use the same
`nextSeq()` + localStorage + upsert-by-id pattern and are therefore exposed to the same
silent-overwrite risk. This is not hypothetical — payroll data entry is actively happening in
this app on an ongoing basis (biweekly runs), so this should ship as a priority, not "eventually."

## Confirmed live schema (as of 2026-09-17)

```
payroll_runs            (id, run_number, check_date, pay_period_start, pay_period_end,
                          pay_frequency, processor, run_type, total_cash_debited, notes, created_at)

payroll_run_employees   (id, payroll_run_id, employee_id, regular_hours, regular_rate,
                          regular_amount, tips_owed_amount, gross_amount, net_pay,
                          peerspace_amount, created_at)

payroll_run_line_items  (id, payroll_run_id, employee_id, category, code, ee_or_er,
                          amount, created_at)
```

Current max IDs in production (confirmed by direct query): `payroll_runs` → `PYRUN-0005`,
`payroll_run_employees` → `PRE-0049`, `payroll_run_line_items` → `PRLI-0493`. The sequences
below are seeded to start one past these.

## Part 1: DB-authoritative IDs (same fix as employees, applied to all 3 tables)

```sql
CREATE SEQUENCE IF NOT EXISTS payroll_runs_id_seq START WITH 6;              -- next free after PYRUN-0005
CREATE SEQUENCE IF NOT EXISTS payroll_run_employees_id_seq START WITH 50;    -- next free after PRE-0049
CREATE SEQUENCE IF NOT EXISTS payroll_run_line_items_id_seq START WITH 494;  -- next free after PRLI-0493

CREATE OR REPLACE FUNCTION public.next_payroll_run_id()
RETURNS TEXT LANGUAGE sql AS $$
  SELECT 'PYRUN-' || lpad(nextval('payroll_runs_id_seq')::text, 4, '0');
$$;

CREATE OR REPLACE FUNCTION public.next_payroll_run_employee_id()
RETURNS TEXT LANGUAGE sql AS $$
  SELECT 'PRE-' || lpad(nextval('payroll_run_employees_id_seq')::text, 4, '0');
$$;

CREATE OR REPLACE FUNCTION public.next_payroll_run_line_item_id()
RETURNS TEXT LANGUAGE sql AS $$
  SELECT 'PRLI-' || lpad(nextval('payroll_run_line_items_id_seq')::text, 4, '0');
$$;
```

In `brew_culture_ap.html`, find the payroll-run save path (same shape as `saveVendor` /
`saveEmployee` — look for wherever `nextSeq('PYRUN')`, `nextSeq('PRE')`, and `nextSeq('PRLI')`
are called, likely in the function that submits a payroll run from the payroll entry screen).
Add matching async ID getters:

```javascript
async function nextPayrollRunId(){
  const {data, error} = await _sb.rpc('next_payroll_run_id');
  if(error){ console.error('nextPayrollRunId error', error); throw error; }
  return data;
}
async function nextPayrollRunEmployeeId(){
  const {data, error} = await _sb.rpc('next_payroll_run_employee_id');
  if(error){ console.error('nextPayrollRunEmployeeId error', error); throw error; }
  return data;
}
async function nextPayrollRunLineItemId(){
  const {data, error} = await _sb.rpc('next_payroll_run_line_item_id');
  if(error){ console.error('nextPayrollRunLineItemId error', error); throw error; }
  return data;
}
```

## Part 2: True insert for new rows, upsert only on edit

Wherever the payroll run is currently saved with `nextSeq(...)` + `.upsert(arr, {onConflict:'id'})`,
split it exactly like the employee fix:

```javascript
// payroll_runs — new run
const run = {id: await nextPayrollRunId(), ...runData};
const ok = await sbInsert('payroll_runs', run);   // TRUE insert, fails loudly on collision
if (!ok) { showError('Could not create payroll run — ID collision, please retry.'); return; }

// payroll_run_employees — one row per employee in the run (new, not an edit)
for (const empData of employeeRows) {
  const row = {id: await nextPayrollRunEmployeeId(), payroll_run_id: run.id, ...empData};
  const ok = await sbInsert('payroll_run_employees', row);
  if (!ok) { showError('Could not save payroll employee row — ID collision, please retry.'); return; }
}

// payroll_run_line_items — same pattern
for (const lineData of lineItemRows) {
  const row = {id: await nextPayrollRunLineItemId(), payroll_run_id: run.id, ...lineData};
  const ok = await sbInsert('payroll_run_line_items', row);
  if (!ok) { showError('Could not save payroll line item — ID collision, please retry.'); return; }
}
```

If there is a genuine **edit** path (correcting an already-saved payroll run/row), keep that on
`sbUpsert` — only the "brand new row" path needs to move to `sbInsert`, exactly as with employees.

## Part 3: Extend the audit log to these tables

If `claude/employee_id_collision_and_audit_log_fix_spec.md` Part 2 has already been applied, the
`audit_log` table and `public.log_audit_event()` function already exist — skip straight to the
`CREATE TRIGGER` statements below. If not, run this whole block first (it's idempotent either way):

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
DROP POLICY IF EXISTS audit_log_admin_read ON public.audit_log;
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
```

Now add the three new triggers (this is the part that's actually new in this addendum):

```sql
DROP TRIGGER IF EXISTS trg_audit_payroll_runs ON public.payroll_runs;
CREATE TRIGGER trg_audit_payroll_runs
AFTER INSERT OR UPDATE OR DELETE ON public.payroll_runs
FOR EACH ROW EXECUTE FUNCTION public.log_audit_event();

DROP TRIGGER IF EXISTS trg_audit_payroll_run_employees ON public.payroll_run_employees;
CREATE TRIGGER trg_audit_payroll_run_employees
AFTER INSERT OR UPDATE OR DELETE ON public.payroll_run_employees
FOR EACH ROW EXECUTE FUNCTION public.log_audit_event();

DROP TRIGGER IF EXISTS trg_audit_payroll_run_line_items ON public.payroll_run_line_items;
CREATE TRIGGER trg_audit_payroll_run_line_items
AFTER INSERT OR UPDATE OR DELETE ON public.payroll_run_line_items
FOR EACH ROW EXECUTE FUNCTION public.log_audit_event();
```

From this point forward, every write to any of these three tables — from the app or from the
Supabase SQL editor — is captured in `audit_log`, so a future collision (even if Part 1/2 above
were somehow bypassed) would leave a visible trail instead of requiring the multi-hour forensics
this incident took.

## Verification after implementing

1. Confirm the sequences start above the real max IDs:
   ```sql
   SELECT nextval('payroll_runs_id_seq'), nextval('payroll_run_employees_id_seq'), nextval('payroll_run_line_items_id_seq');
   ```
   then immediately reset them back down by one with `setval(..., currval(...) - 1)` if this is
   run before any real new row is created, so the first real save gets the intended next ID.
2. Enter one test payroll run end-to-end (a run, at least one employee row, at least one line
   item) and confirm each gets a fresh `PYRUN-`/`PRE-`/`PRLI-` ID rather than colliding with an
   existing row.
3. Confirm `audit_log` picked up INSERT rows for the run, the employee row(s), and the line
   item(s), each with the correct `new_data` JSON and no `old_data`.
4. Delete the test run/rows and their audit_log entries once confirmed.
5. Re-run this spot check on `payroll_run_employees` periodically until this ships (particularly
   after any payroll entry session), since the underlying bug is still live in production until
   Part 1/2 above are deployed:
   ```sql
   SELECT count(*), max(id) FROM payroll_run_employees;
   SELECT count(*), max(id) FROM payroll_run_line_items;
   SELECT count(*), max(id) FROM payroll_runs;
   ```
   A jump in count with the previously-repaired employees' (EMP-0015/EMP-0016/EMP-0017) rows
   unchanged is the expected healthy outcome; any change to an *existing* row's `employee_id` or
   dollar amounts without a corresponding new row would indicate another overwrite.
