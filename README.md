# DCPR – Data Capture Project Requests (Form E1 / Form E2)

This Django app digitises the two CSI forms and the NSIF_B TD.5 process:

* **Form E1** is a non-custodian request for CSI approval to capture spatial data.
* **Form E2** is a Base Dataset Custodian notification of intention to capture spatial data.

## Quick start

```bash
python -m venv .venv && source .venv/bin/activate      # Windows: .venv\Scripts\activate
pip install -r requirements.txt
python manage.py migrate          # creates role groups and loads SA boundaries (~15 s)
python manage.py seed_demo        # optional demo users, password: demo12345
python manage.py runserver
```

Open http://localhost:8000. Demo users: `applicant`, `nsif`, `committee`, `chair`, `admin` (superuser).
Run the tests with `python manage.py test dcpr`.

## Roles (Django groups, assign in /admin)

| Group | Does |
|---|---|
| *(any registered user)* | Completes Form E1/E2, attaches documents, tracks progress |
| NSIF (Spatial Information and Support) | Acknowledges, assesses, prepares adjudication pack, notifies, scans & files |
| CSI Data Sub-committee | Decides whether the request meets the criteria (Y/N) |
| CSI Data Sub-committee Chairperson | Signs the outcome letter, uses the CSI feedback report |

## Workflow (dcpr/workflow.py)

Form E1:
`Draft → Submitted → Acknowledged (register updated) → With Sub-committee (assessment + adjudication pack)
→ Awaiting Chairperson signature (outcome: successful/unsuccessful) → Letter signed → Applicant notified
→ Filed & closed (register updated)`

Form E2 (Base Dataset Custodian notification) follows **the same process** as Form E1: acknowledge, assess,
Sub-committee decision, Chairperson's letter, notify, and scan & file, including refer-back. Everything is
worded as a *notification* instead of a *request*: the action labels, help text, progress, history, emails
and messages.

**Refer back for amendments.** The role that holds a request can refer it back to the applicant.
It must give a reason, which is emailed to the applicant, and can attach marked-up documents.

| Stage | Who can refer back |
|---|---|
| Submitted / Acknowledged | NSIF |
| With CSI Data Sub-committee | CSI Data Sub-committee |
| Awaiting Chairperson signature | Chairperson |

The applicant amends the form (datasets and documents included) and resubmits it.
They must accept the declaration again. The resubmitted request keeps its reference number and goes back to
NSIF at "Submitted". If you prefer that any of the three roles can refer back at any stage, edit
`roles_by_status` on the `refer_back` transition in `dcpr/workflow.py`.

Each step is logged in an audit trail, and the relevant people are emailed. Email goes to the console backend
until you configure SMTP in `config/settings.py`.

## Geographic extent and spatial search

Each dataset has a map picker, and its extent can be set in several ways. The maps are drawn by DCPR's own
SVG map (`static/dcpr/dcpr-map.js`) from the boundary data in the database. There are no basemap tiles, CDNs or
external libraries, so the maps work on closed or offline networks. The country and provinces are always drawn
for orientation, and names appear as you zoom. You can pan and zoom with a mouse, trackpad, touch, keyboard
or the on-map buttons, and a scale bar is shown.

* **Predefined boundaries:** type to search provinces, districts/metros and local municipalities, or click
  them on the map. You can choose several, at any level.
* **A new boundary:** draw a polygon, draw a bounding box, type in bounding-box coordinates, or upload GeoJSON.
* **Both:** the extent is the union of everything selected. It is stored with an indexed bounding box and its area.

Every dataset needs an extent. The paper form's tick-boxes are kept as "Geographic extent level".

**Spatial search** (`/search/`, office roles) finds datasets that overlap, or lie completely within, a chosen
area. The area can be boundaries, a drawn shape or a bounding box, and it combines with text, form, status,
data-type and method filters. The results appear on a map and in a table.

Each request page also lists **overlapping capture requests** for NSIF and the committee, with the overlap area,
to help spot duplicate capture during the preliminary assessment. The register CSV includes each dataset's
areas, area in km² and bounding box. The CSI report counts datasets per province, and the printed form shows a
small locator map for each dataset.

### Boundary data

`migrate` loads five bundled layers (see `dcpr/data/README.md`), which link into one hierarchy by official
MDB code:

| Level | Count | Source |
|---|---|---|
| Country | 1 | OCHA COD-AB South Africa (MDB / Stats SA, valid 2020-11-09) |
| Province | 9 | OCHA COD-AB |
| District / metro | 52 | **MDB District Municipal Boundaries 2018** |
| Local / metro municipality | 213 | **MDB Local Municipal Boundaries 2018** |
| Ward | 4 392 | OCHA COD-AB |

In the picker, you can search any level, including wards ("Stellenbosch Ward 12"), or click boundaries on the
map. Wards load for the area on screen once you zoom in. A "Whole of South Africa" button selects the national
boundary.

### Uploading or updating boundaries (administrators)

Go to **Admin › Boundary dataset uploads › Add**, then:

1. **Upload the file.** Choose the level and upload GeoJSON (WGS84), a .gz of it, or a zipped shapefile.
   Original MDB and OCHA COD-AB files work as-is, because their fields are detected automatically. For other
   layouts, set the field mapping or a name pattern such as `{MUNICNAME} Ward {WARDNO:int}`.
2. **Validate (dry run).** Every feature is checked: coordinate system, name and code, duplicate codes,
   polygon geometry and location inside South Africa. The result shows what would change, without writing
   anything.
3. **Import.** Boundaries are matched on their code within the level and updated in place.
   * *Update* adds new boundaries and updates existing ones.
   * *Replace* also retires boundaries that are missing from the file.
   * Retired boundaries stay attached to the requests that used them, and those requests keep their saved
     extents. They are hidden from the picker.
   * Parents link by code, so load provinces before districts, districts before municipalities, and
     municipalities before wards. The bulk "Import selected" action does this order for you.

Each upload keeps its file, the counts, a log and which boundaries came from it. A district file can also
rebuild provinces by merging districts on their province code ("Derive provinces").

To let NSIF staff upload boundaries without full admin rights, give them staff status and the "Can add/change
boundary dataset upload" permissions.

Command line: `python manage.py load_boundaries <file> --level municipality [--dry-run]`, or `--bundled` to
reload the shipped data.

**Large files:** importing the full 162 MB ward file takes about 30 seconds. Raise the app server timeout
(e.g. `gunicorn --timeout 120`) and the proxy upload limit (e.g. nginx `client_max_body_size 200m`).
`DCPR_BOUNDARY_TOLERANCE` (default 0.0005°, about 50 m) controls how much uploads are simplified; set it to 0
to keep full resolution.

### How spatial queries work

Geometries are stored as GeoJSON in JSON columns, each with indexed bounding-box columns. A search narrows the
candidates by bounding box in SQL, then tests the exact geometry with Shapely (`dcpr/geo.py`). This runs on
SQLite and PostgreSQL without GDAL or PostGIS. For very large volumes you could move to GeoDjango and PostGIS
without changing the views.

### Maps without external connections

`DCPR_MAP` in `config/settings.py` only holds the attribution text. The maps never contact another server.
To show more context (roads, rivers and so on), load it as another boundary level through the admin.

## Datasets

* A project can produce several datasets. The form has an "Add another dataset" button, and each dataset has its
  own description, spatial characteristics and method of capture.
* Geographic extent, data type and method of capture are multi-select. Each needs at least one tick.
  "Other" geographic extent asks for a description.
* The register CSV has one row per dataset, and the print view repeats the dataset block for each dataset.

## Supporting documentation (optional)

* Supporting documents are **not** mandatory. A document row left without a file is ignored.
* Applicants can attach any number of files on the form (detailed specification, project plan, ToR, sample data,
  budget, other). They can keep adding documents after submission until adjudication starts.
* Office steps attach their own documents (acknowledgement letter, adjudication pack, signed letter, scanned file).
* Files are stored under `MEDIA_ROOT` and are **only** served through a permission-checked download view.
  Applicants never see internal documents, and they see the signed letter only once they have been notified.
* The allowed extensions and the 25 MB size limit are set in `config/settings.py`
  (`ATTACHMENT_ALLOWED_EXTENSIONS`, `ATTACHMENT_MAX_SIZE_MB`).

## Other features

* **Register** (`/register/`): filterable, with CSV export. This is the Spatial Data Capture Project Register.
* **CSI feedback report** (`/report/`): totals, values and breakdowns, and it prints to PDF.
* **Print view**: a paper-style Form E1/E2 with the selected options marked X.
* **Reference numbers**: `CSI/E1/2026/0001`, allocated on submission.

## Changes from the draft model

* Fixed fields that were never created: `email = models.EmailField`, `startDate = models.DateField`, and the same
  for `endDate` and `estCost`, were missing `()`.
* `dataseType` had `max_length=1` but a choice `"Po"`. Data type, geographic extent and method of capture are
  now multi-select lists (`data_types`, `geographic_extents`, `capture_methods`, stored as JSON).
  Migration 0003 converts any existing single values.
* Renamed the `ogranisationContact` typo. Fields now use snake_case.
* Fields that were on the forms but missing from the model: Tribal Authority level, fax, urgency, data usage
  restrictions, feature description, Land parcels / Other geographic extent, declaration.
* Free-text fields became `TextField`, which removes the 255-character limit. Cost is now `DecimalField` (ZAR).
* Added `CaptureRequest` (the submission plus its status), `Attachment` and `WorkflowEvent`.

## Branding

The header carries the Department of Land Reform and Rural Development logo and the CSI logo
(`static/dcpr/img/`). The colours follow the department's brown, gold and green (CSS variables at the top of
`static/dcpr/style.css`). The printed form carries both logos.

## Production notes

Set `DJANGO_SECRET_KEY`, `DJANGO_DEBUG=0`, `DJANGO_ALLOWED_HOSTS` and SMTP settings, and use PostgreSQL.
Run `collectstatic` and put `MEDIA_ROOT` on backed-up storage.
