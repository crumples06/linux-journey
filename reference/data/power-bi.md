# Power BI

Notes from building a report for work from an Excel sheet. (Not Linux, kept here as its own reference.)

## The real work is data shaping

The effort is not the chart-making; it is getting data into a shape that is easy to chart. Often you build whole intermediate tables just to support one visual.

## Slicers and relationships

- A **slicer** lets report viewers filter by a value.
- By default a slicer only affects visuals built from the **table the slicer came from**. Visuals built from other tables are unaffected, even if those tables share the same values.
- Fix: open **Model view** and create relationships.
  1. Make a new table containing the distinct values the tables have in common.
  2. Relate each data table to it with a **many-to-one** relationship.
  3. Build the slicer from the new common table.
  Now the slicer filters every visual connected through it.
- General pattern: a small shared lookup/dimension table in the middle, fact tables on the "many" side.

## Making it look good quickly

Rather than formatting every visual by hand, import a premade **theme** (downloaded from Microsoft's site). One import restyles the whole report consistently.
