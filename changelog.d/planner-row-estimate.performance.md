- A non-key equality now prices one row in 100 and a hop's estimate may pass
  the destination count; the planner picks hash joins, CSR builds and key-set
  blocks from them. See the [explain guide][planner-row-estimate-explain].

[planner-row-estimate-explain]: ../docs/user/queries/explain.md
