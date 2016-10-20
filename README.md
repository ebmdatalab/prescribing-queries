# Prescribing queries

Documentation about common queries against the prescribing dataset.

## Overview of datasets in bigquery

* `hscic:prescribing`: the main prescribing dataset. Updated monthly
* `hscic:practices`: all the practices in England. Updated monthly. Practices with a `setting` of `4` are standard GP practices
* `hscic:praactice_statistics`: total list size, STAR-PU, ASTRO-PU and list sizes stratified by gender and age group for each practice. Updated monthly.
* `hscic:presentation`: ADQs and related data for each BNF code. Currently updated manually as needed

## Common queries

Below are some example queries. As we collect more real-life queries, we should document them as issues [here](https://github.com/ebmdatalab/prescribing-queries/issues) and link to notable examples from this document.

### Compute ratios - BNF code in numerator and denominator

Note that this query includes a join to the `practices` table, such that we return a row for every practice in every month, even where there is no data for that month/practice combination. This is a common requirement.

Also note the condition restricting the results to practices where the `setting` is `4` -- these are "standard GP practices" and this is the default we present in OpenPrescribing.  Some users may want data across the whole NHS.

**WARNING** this query will return a *lot* of rows!

```sql

SELECT
  -- Compute ratios
  *,
  IEEE_DIVIDE(numerator, denominator) AS calc_value
FROM (
  SELECT
    *
  FROM (
    SELECT
      COALESCE(num.numerator,
        0) AS numerator,
      COALESCE(denom.denominator,
        0) AS denominator,
      practices_with_months.practice_id,
      practices_with_months.ccg_id AS ccg_id,
      practices_with_months.month
    FROM (
      SELECT
        month,
        practice,
        SUM(items) AS numerator
      FROM
        ebmdatalab.hscic.prescribing
      WHERE
        bnf_code LIKE '0212000AA%'
      GROUP BY
        practice,
        month) num
    RIGHT JOIN (
      SELECT
        month,
        practice,
        SUM(items) AS denominator
      FROM
        ebmdatalab.hscic.prescribing
      WHERE
        bnf_code LIKE '0212%'
      GROUP BY
        practice,
        month) denom
    ON
      (num.practice=denom.practice
        AND num.month=denom.month)
    RIGHT JOIN (
        -- Return a row for every practice in every month, even where
        -- there is no denominator value
      SELECT
        prescribing.month AS month,
        practices.code AS practice_id,
        ccg_id
      FROM
        `ebmdatalab.hscic.practices` AS practices
      CROSS JOIN (
        SELECT
          month
        FROM
          ebmdatalab.hscic.prescribing
        GROUP BY
          month) prescribing
      WHERE
        practices.setting = 4) practices_with_months
    ON
      practices_with_months.practice_id = denom.practice
      AND practices_with_months.month = denom.month
  )
)
```

### Compute ratios for single practice

Wrap the above query with an outer query, like this:

```sql

SELECT * FROM (
 SELECT
  -- Compute ratios
  *,
  IEEE_DIVIDE(numerator, denominator) AS calc_value

  ...
    ON
      practices_with_months.practice_id = denom.practice
      AND practices_with_months.month = denom.month
    )
  )
)

WHERE practice_id = 'P82037'
```
### Compute ratios, group by CCG

Wrap the above query with an outer query, like this:
```sql

SELECT
  month,
  ccg_id,
  sum(numerator)/sum(denominator) AS calc_value
FROM (
 SELECT
  -- Compute ratios
  *,
  IEEE_DIVIDE(numerator, denominator) AS calc_value

  ...
    ON
      practices_with_months.practice_id = denom.practice
      AND practices_with_months.month = denom.month
    )
  )
)

GROUP BY month, ccg_id
ORDER BY month
```

## Other examples

* [Diabetes drugs per list size, grouped by year and CCG](https://github.com/ebmdatalab/prescribing-queries/issues/1#issuecomment-255042978)
* [Diabetes drugs: total actual cost per month, grouped by CCG, standard practices only](https://github.com/ebmdatalab/prescribing-queries/issues/1#issuecomment-255061302)

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
