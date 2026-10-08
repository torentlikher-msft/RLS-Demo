# RLS and Copilot: Row-Level Security Design Demo

A Power BI demo that compares ways of implementing **department-based row-level security (RLS)** on a healthcare Patient Encounters model, and measures what each design costs in **capacity CPU** (what Fabric bills).

The models are also described in detail (tables, columns, measures and roles) so they work well with **Copilot** and **Fabric data agents**.

> All data is synthetic and generated inside Power Query. No external data sources or credentials are needed.

---

## What's in this repo

Each folder pair is a [Power BI Project (PBIP)](https://learn.microsoft.com/power-bi/developer/projects/projects-overview). Open the `.pbip` file in Power BI Desktop.

| Project | RLS design | Scenarios |
|---|---|---|
| `RLS and Copilot Active Relationship.pbip` | One-line rule on the security table, which reaches Department through an **active, two-way relationship** with security filtering in both directions | 1 |
| `RLS and Copilot Disconnected Tables.pbip` | Security tables **not related** to Department; one role per design, each with a DAX rule on Department | 2, 3, 4, 5 |
| `RLS and Copilot All Access Role.pbip` | Normalized DAX rule for users with partial access, plus a separate **All Access** role with no data filters | 6 |

All three share the same star schema, measures and report page (**RLS Test**), and differ only in security tables, relationships and roles.

---

## The model

### Star schema

| Table | Type | Notes |
|---|---|---|
| `Encounters` | Fact | One row per patient encounter. Allowed amount, length of stay, readmission flag |
| `Date` | Dimension | Calendar 2022–2026 |
| `Department` | Dimension | 2,500 rows: 10 clinical departments at each of 250 facilities (department, service line, facility name, type, city, state) |
| `Patient` | Dimension | One patient per 100 encounters; age band, sex, race and ethnicity, cohort |
| `Provider` | Dimension | 5,000 providers, with specialty and type |
| `Payer` | Dimension | Payer name and type |
| `Diagnosis` | Dimension | 100 diagnoses in clinical categories |
| `Encounter Type` | Dimension | 8 visit types (for example Inpatient, Emergency, Observation, Outpatient, Telehealth) grouped into care settings |
| `Discharge Disposition` | Dimension | Includes Expired, used by the mortality measures |
| `_Measures` | Measures | 30 measures: volume, financial, length of stay, quality (readmission, mortality), time intelligence (YTD, LY, YoY) and RLS helpers (`Current User`, `Visible Departments`) |

### Data volume

The **`Encounter Row Count`** parameter controls the size of the fact table. The projects in this repo are set to **10,000** rows so they open and refresh quickly.

To scale up, change the parameter in **Transform data → Manage parameters** and refresh. The benchmarks used **50,000,000** rows.

The data is generated in chunks from a fixed random seed, so the same value always produces the same data. Very large values increase refresh time and memory use.

### Security tables

All designs give **2,006 synthetic users** the same access to the 2,500 departments. They differ only in how the access list is stored.

| Table | Shape |
|---|---|
| `User Department Access` | **Normalized**: one row per user per department (about 240,000 rows) |
| `User Department Access (Override)` | Normalized, plus an `All Access` flag; all-access users need a single row (about 112,000 rows) |
| `User Department Access (Delimited)` | One row per user, with a pipe-delimited key list such as `11\|12\|13` (2,006 rows) |
| `User Department Access (Delimited Override)` | Delimited list plus an `All Access` flag; all-access users have an empty list (2,006 rows) |

Every role also limits each security table to the signed-in user's own row, so nobody can read other users' access lists.

---

## The six RLS scenarios

| # | Scenario | Role rule | Project |
|---|---|---|---|
| 1 | Normalized + relationship | `'User Department Access'[User Email] = USERPRINCIPALNAME()`; a two-way security relationship carries the filter to Department | Active Relationship |
| 2 | Normalized | DAX rule on `Department`: the user's granted keys from the normalized table | Disconnected Tables (`Department Access Normalized`) |
| 3 | Normalized + override | As scenario 2, plus an All Access flag check | Disconnected Tables (`Department Access Normalized Override`) |
| 4 | Delimited | `PATHCONTAINS` over the user's key string | Disconnected Tables (`Department Access Delimited`) |
| 5 | Delimited + override | As scenario 4, plus an All Access flag check | Disconnected Tables (`Department Access Delimited Override`) |
| 6 | Normalized + All Access role | Scenario 2's rule for partial-access users; a separate `All Access` role with **no** filter on Department | All Access Role 



## Getting started

1. **Open** a `.pbip` file in Power BI Desktop (a current version, with PBIP and TMDL support).
2. **Refresh** to generate the data, optionally after changing `Encounter Row Count`.
3. **Test RLS in Desktop.** Go to **Modeling → View as**, choose a role, tick **Other user** and enter one of the synthetic users:

   | User | Departments visible |
   |---|---|
   | `executive@contoso.com` | 2,500 (all) |
   | `specialty.analyst@contoso.com` | 1,000 |
   | `regional.leader.0032@contoso.com` | 250 |
   | `facility.director.0001@contoso.com` | 10 |
   | `nobody@contoso.com` | none |

   The **RLS Test** page shows the current user, visible departments and key totals.
4. **Publish** to a Fabric or Premium workspace.
5. **Assign role members** on the semantic model's **Security** page. In the service, `USERPRINCIPALNAME()` returns real sign-in addresses, so test users must exist in the security tables. Workspace Admins, Members and Contributors bypass RLS, so test with users who only have **Read** (Viewer) access.

### Copilot and data agents

Every table, column, measure and role has a detailed description for AI use. To test a Fabric data agent with RLS:
- Give the test user **Read** access to the semantic model (not a workspace role that bypasses RLS).
- Add the test user to the security tables and the right role.
- Sign in as that user and ask the agent questions.

---

## Key findings

Tested in the **Power BI service on an F64 capacity** at **50 million encounters**, with four users who can see 2,500, 1,000, 250 and 10 departments. CPU comes from Fabric workspace monitoring (`QueryEnd` CPU time, all engine threads).

| Scenario | CPU, report session* | CPU, one-off queries** |
|---|---|---|
| 2. Normalized | +1% | best |
| 3. Normalized + override | +1% | +4% |
| 4. Delimited | +2% | +3% |
| 5. Delimited + override | +2% | +3% |
| 1. Normalized + relationship | **+15%** | **+105%** |

\* Average gap to the best design across the four users; a report session is 19 visuals (page load, slicer click, second page).
\*\* 7 queries, each on a new connection with a cleared cache, like a Copilot or data-agent question.


- **Scenarios 2–5 cost about the same CPU**, within 5% of each other.
- **An All Access role (scenario 6) removes the RLS cost for all-access users.** Their sessions ran at no-RLS cost, about 21% less CPU than under scenario 2's rule.
- **Capacity size matters more than RLS design.** On an F8 capacity the benchmark triggered Fabric throttling, which delayed every request by about 20 seconds.

### Recommendations

1. **Use a normalized security table with a DAX rule on Department** (scenario 2 or 3), and leave the security table unrelated to Department.
2. **Avoid the two-way security relationship** (scenario 1). It's the simplest rule but the most expensive in capacity, and it stops any role from filtering Department directly.
3. **Give all-access users their own role with no Department filter** (scenario 6), managed through a tightly controlled Microsoft Entra security group. Add an own-row filter on the security table, `'User Department Access'[User Email] = USERPRINCIPALNAME()`, so members can't read other users' access lists. The `All Access` role in this repo doesn't have that filter yet.
4. **Prefer normalized rows to delimited strings.** If access arrives as strings, split it into rows in Power Query.

### Caveats
- Performance differs when testing in Power BI Desktop vs the service. Ensure that no throttling is taking place if testing in the service.
- CPU Utilization heavily depends on data volume and # of total departments.
- Synthetic data and generated users; one test user at a time with no concurrent load.
- Visuals ran one after another, whereas Power BI runs several at once.

---

## Repository notes

- `.pbi/cache.abf` files hold a small local data cache (about 0.5 MB at 10,000 rows) so the projects open with data. They're regenerated on refresh. If you scale the row count up, don't commit the larger cache.
