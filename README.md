# Prescribing queries

Documentation about common queries against the prescribing dataset.

## Using BigQuery

You will need a Google account with the correct permissions (set up by an administrator) to access our BigQuery account.

Then follow [this quickstart](https://cloud.google.com/bigquery/quickstart-web-ui) (ignoring the first section "Before you begin")

### Legacy vs Standard

BigQuery provides a SQL-like interface to massive datasets.  It has two dialects, "Legacy" and "standard". When running a query, the default is "Legacy"; you must select "standard" in the options section to use that.  Nearly all the examples here are in "standard" format, which is compatible with standard SQL. However, sometimes it is necessary to use "Legacy" format as some functions have not yet been ported by Google to the newer format.

* [Standard SQL documentation](https://cloud.google.com/bigquery/docs/reference/standard-sql/)
* [Legacy SQL documentation](https://cloud.google.com/bigquery/docs/reference/legacy-sql)

Standard SQL supports [temp tables](https://cloud.google.com/bigquery/docs/reference/standard-sql/query-syntax#with_query_name) which can make your queries more readable than using lots of subqueries.

Legacy SQL used to have a more extensive range of [aggregate functions](https://cloud.google.com/bigquery/docs/reference/legacy-sql#functions), and in particular, [window functions](https://cloud.google.com/bigquery/docs/reference/legacy-sql#syntax-window-functions).  However, OpenPrescribing is now able to use standard SQL exclusively.

A comparison between the two formats is [here](https://cloud.google.com/bigquery/docs/reference/standard-sql/migrating-from-legacy-sql#comparison_of_legacy_and_standard_sql).

### Billing

BigQuery is billed by the amount of data queried. Querying the entire prescribing table costs about 20 cents. You should bear these costs in mind if running large numbers of queries. Good practice is to extract, say, one month of data to a new table to design your queries, e.g. running

```sql
SELECT *
FROM ebmdatalab.hscic.prescribing
WHERE month = TIMESTAMP('2016-06-01')
```
...and selecting "save to table". If you then save this to a table `ebmdatalb.tmp.<something>`, then you can continue to design your query like:

```sql
SELECT *
FROM ebmdatalab.tmp.<something>
WHERE bnf_code LIKE '02%'
LIMIT 1000
```

### Overview of tables in BigQuery

* `hscic.prescribing`
  * The main prescribing dataset.
  * Updated monthly. Data going back to Aug 2010.
  * Data from https://apps.nhsbsa.nhs.uk/infosystems/data/showDataSelector.do?reportId=124
  * You probably want to query one of the `normalised_prescribing_*` tables, described below
  * Contains one row per practice per presentation per month
  * Fields:
    * `sha`: Area team code (TODO: what is this?)
    * `pct`: Identifier of CCG
    * `practice`: Identifier of practice
    * `bnf_code`: BNF code of presentation
    * `bnf_name`: Name of presentation in BNF
    * `items`: Number of items, where "item" means "appearance on a prescription"
    * `net_cost`: The cost of the presentation that month to that practice, according to the Drug Tariff
    * `actual_cost`: The actual cost when taking into account adjustments for bulk purchases, out of pocket expenses, etc. This is the "cost" field that we usually want to query
    * `quantity`: number of pills/grams/millilitres/dressings/ampoules prescribed
    * `month`: Month, as a `TIMESTAMP` of the first millisecond of the month

* `hscic.normalised_prescribing_standard` and `hscic.normalised_prescribing_legacy`
  * Views on the `prescribing` table that normalise the `pct` and `bnf_code` fields as described below.
  * In most cases, this is the table you will want to query.
  * The data is the same in both tables.  Use `normalised_prescribing_standard` when you want to use Standard SQL, and `normalised_prescribing_legacy` when you want to use Legacy SQL.
  * Fields
    * As for the `prescribing` table, except:
      * The `pct` field is renamed to `ccg_id`, and is the identifier of the CCG that the practice is _currently_ in, as it may have moved between CCGs
      * The `bnf_code` field is the most recent version of the BNF code

* `hscic.practices`:
  * All the practices in England.
  * Updated quarterly.
  * Data from https://digital.nhs.uk/services/organisation-data-service/data-downloads/gp-and-gp-practice-related-data
  * Practices with a `setting` of `4` are standard GP practices (see below for a full list of `settings`s)
  * Fields:
    * `code`: Identifier of practice
    * `name`: Name of practice
    * `address1` to `address5`, `postcode`: Portions of address
    * `location`: Location of practice in WG84 format
    * `ccg_id`: Identifier of practice's current CCG
    * `setting`: See below for a full list of `settings`s
    * `close_date`: Date practice closed
    * `join_provider_date`: Date practice joined CCG
    * `leave_provider_date`: Date practice closed
    * `open_date`: Data practice opened
    * `status_code`: One of the following:
        *`A`: Active
        *`B`: Retired
        *`C`: Closed
        *`D`: Dormant
        *`P`: Proposed
        *`U`: Unknown

* `hscic.ccgs`:
  * All the CCGs in England.
  * Updated quarterly.
  * Data from https://digital.nhs.uk/services/organisation-data-service/data-downloads/other-nhs-organisations
  * Fields:
    * `code`: Identifier of CCG
    * `name`: Name of CCG
    * `ons_code`: ONS code
    * `org_type`: One of the following:
      * `CCG`
      * `PCT`
      * `Unknown`
    * `open_date`: Date CCG was formed
    * `close_date`: Date CCG closed
    * `address`, `postcode`: Address of principal office

* `hscic.practice_statistics`:
  * Total list size, STAR-PU, ASTRO-PU, and list sizes stratified by gender and age group for each practice.
  * Updated monthly.
  * Data from https://digital.nhs.uk/data-and-information/publications/statistical/patients-registered-at-a-gp-practice
  * Fields:
    * `month`
    * `practice`
    * `pct_id`
    * `total_list_size`
    * `astro_pu_cost`
    * `astro_pu_items`
    * `star_pu`
    * For each age group in :`0_4`, `15_24`, `25_34`, `35_44`, `45_54`, `55_64`, `5_14`, `65_74`, `75_plus`
        * `female_{age_group}`
        * `male_{age_group}`

* `hscic.presentation`:
  * BNF code, name, ADQs for each presentation.
  * BNF data:
    * Updated monthly
    * Data from https://apps.nhsbsa.nhs.uk/infosystems/data/showDataSelector.do?reportId=126
  * ADQs updated manually when new data available.
  * Fields:
    * `bnf_code`
    * `name`
    * `is_generic`
    * `active_quantity`
    * `adq`
    * `adq_unit`
    * `percent_of_adq`

* `hscic.ppu_savings`:
  * Price per Unit savings
  * Updated monthly based on prescribing data
  * Fields:
    * `date`
    * `pct_id`
    * `practice_id`
    * `bnf_code`
    * `lowest_decile`
    * `quantity`
    * `price_per_unit`
    * `possible_savings`
    * `formulation_swap`

* `dmd.product`:
  * Products (both AMPs and VMPS) in dm+d
  * Updated monthly
  * Data from https://isd.digital.nhs.uk/trud3/user/authenticated/group/0/pack/6/subpack/24/releases
  * See https://www.nhsbsa.nhs.uk/sites/default/files/2017-02/dmd_Implemention_Guide_%28Primary_Care%29_v1.0.pdf for details of fields

* `dmd.tariffprice`:
  * Monthly prices according to the Drug Tariff
  * Updated monthly
  * Data from https://www.nhsbsa.nhs.uk/pharmacies-gp-practices-and-appliance-contractors/drug-tariff/drug-tariff-part-viii/
  * Fields:
    * `date`
    * `vmpp`
    * `product`
    * `tariff_category`
    * `price_pence`

* `dmd.ncsoconcession`:
  * Temporary alterations to official Drug Tariff prices in response to things like shortages, etc
  * Updated monthly, or when new data is available
  * Data from http://psnc.org.uk/dispensing-supply/supply-chain/generic-shortages/ncso-archive/
  * Fields:
    * `vmpp`
    * `date`
    * `drug`
    * `pack_size`
    * `price_concession_pence`

* `dmd.vmpp`
  * Virtual Medicinal Product Packs (part of the dm+d data model, see our [dm+d notes](https://github.com/ebmdatalab/openprescribing/wiki/PPD-data-model#dmd-notes) for more details)
  * Used for linking tariffs and concessions to products
  * Fields:
    * `vvpid`: Primary key, corresponds to `tariffprice.vmpp` and `ncsoconcession.vmpp`
    * `invalid`
    * `nm`
    * `abbrevnm`
    * `vpid`
    * `qtyval`
    * `qty_uomcd`
    * `combpackcd`


## Practice settings

The different kinds of practice available in the `setting` column of the `practices` table are as follows:

* 0 = Other
* 1 = WIC Practice
* 2 = OOH Practice
* 3 = WIC + OOH Practice
* 4 = GP Practice
* 8 = Public Health Service
* 9 = Community Health Service
* 10 = Hospital Service
* 11 = Optometry Service
* 12 = Urgent & Emergency Care
* 13 = Hospice
* 14 = Care Home / Nursing Home
* 15 = Border Force
* 16 = Young Offender Institution
* 17 = Secure Training Centre
* 18 = Secure Children's Home
* 19 = Immigration Removal Centre
* 20 = Court
* 21 = Police Custody
* 22 = Sexual Assault Referral Centre (SARC)
* 24 = Other

## The prescribing table and BNF Codes

The datain the prescribing table covers prescriptions prescribed by
GPs and other non-medical prescribers (nurses, pharmacists and others)
in England and dispensed in the community in the UK. Prescriptions
written in England but dispensed outside England are included.

The format of the data in the prescribing table is documented by the NHS [here](http://content.digital.nhs.uk/media/10686/Download-glossary-of-terms-for-GP-prescribing---presentation-level/pdf/PLP_Presentation_Level_Glossary_April_2015.pdf).

The unique identifier for the item prescribed is the `bnf_code`.

The BNF (British National Formulary) is the de facto standard list of
medicines used UK prescribing. Maintained by the British Medical
Association and the Royal Pharmaceutical Society, it lists
indications, dosages, side effects and so on for more than 70,000
medicines.

The prescribing data uses a modified version of the BNF  which was current in 2014, with custom additions and alterations. The resulting system is called a BNF pseudo-classification and is  described [here](http://www.nhsbsa.nhs.uk/PrescriptionServices/3197.aspx). In particular, appliances are not listed in the BNF at all, so are included in the pseudo-classification with sections named `DUMMY SECTION`.

The first characters of the code provide a hierarchical classification of the presentation.

The last few characters identify individual presentations, and a way
of identifying their generic equivalents.

The image below shows how you might examine Tramadol. Tramadol is an
opiod pain medicine, available in the UK as tablets (i.e. pills),
capsules (i.e. gelatine things), patches, liquids and more.  Just
focussing on tablets, these are available as standard tablets, and
modified release tablets (which are absorbed by the body over a longer
period of time, allowing the patient to take less frequent doses).
Modified release tablets are available in 50mg, 100mg, 150mg, 200mg,
300mg and 400mg pills.

![image](https://raw.githubusercontent.com/ebmdatalab/prescribing-queries/master/bnf_code_understander.png)
