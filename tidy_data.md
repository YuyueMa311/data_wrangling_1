tidy_data
================

This document will show how to tidy data.

## Pivot longer

shorten the columns and combine variables `names_prefix` delete prefix
among all variables changing bl to 00m for consistency across visits and
converting visit to a factor variable

``` r
pulse_df = read_sas("./data_import_examples/public_pulse_data.sas7bdat")|>
  janitor::clean_names()|>
  pivot_longer(
    cols = bdi_score_bl:bdi_score_12m,
    names_to = "visit",
    values_to = "bdi_score",
    names_prefix = "bdi_score_"
  )|>
  mutate(visit = replace (visit, visit == "bl", "00m")) |>
  relocate(id,visit)
```

Learning Assessment Code:

``` r
litters_wide = 
  read_csv(
    "./data_import_examples/FAS_litters.csv",na = c("NA", ".", "")) |>
  janitor::clean_names() |>
  pivot_longer(
    cols = gd0_weight:gd18_weight,
    names_to = "gd_time", 
    values_to = "weight") |>
  mutate(gd_time = case_match(gd_time, "gd0_weight" ~ 0, "gd18_weight" ~ 18))
```

    ## Rows: 49 Columns: 8
    ## ── Column specification ────────────────────────────────────────────────────────
    ## Delimiter: ","
    ## chr (2): Group, Litter Number
    ## dbl (6): GD0 weight, GD18 weight, GD of Birth, Pups born alive, Pups dead @ ...
    ## 
    ## ℹ Use `spec()` to retrieve the full column specification for this data.
    ## ℹ Specify the column types or set `show_col_types = FALSE` to quiet this message.

## Pivot wider

Let’s make up on analysis result table.

``` r
analysis_result = 
  tibble(
    group = c("treatment", "treatment", "placebo", "placebo"),
    time = c("pre", "post", "pre", "post"),
    mean = c(4, 8, 3.5, 4)
  )

analysis_result
```

    ## # A tibble: 4 × 3
    ##   group     time   mean
    ##   <chr>     <chr> <dbl>
    ## 1 treatment pre     4  
    ## 2 treatment post    8  
    ## 3 placebo   pre     3.5
    ## 4 placebo   post    4

Pivot wider for human readability: knitr::kable() produce a nicer table
for reading in html format.

``` r
pivot_wider(
  analysis_result, 
  names_from = "time", 
  values_from = "mean") |>
  knitr::kable()
```

| group     | pre | post |
|:----------|----:|-----:|
| treatment | 4.0 |    8 |
| placebo   | 3.5 |    4 |

## Bind tables.

Import/Read each table from excel:

``` r
fellowship_ring = 
  readxl::read_excel("./data_import_examples/LotR_Words.xlsx", range = "B3:D6") |>
  mutate(movie = "fellowship_ring")

two_towers = 
  readxl::read_excel("./data_import_examples/LotR_Words.xlsx", range = "F3:H6") |>
  mutate(movie = "two_towers")

return_king = 
  readxl::read_excel("./data_import_examples/LotR_Words.xlsx", range = "J3:L6") |>
  mutate(movie = "return_king")
```

Stack tables up, using `bind_rows` function:

``` r
lotr_tidy = 
  bind_rows(fellowship_ring, two_towers, return_king) |>
  janitor::clean_names() |>
  pivot_longer(
    female:male,
    names_to = "gender", 
    values_to = "words") |>
  mutate(race = str_to_lower(race)) |> 
  select(movie, everything()) 

lotr_tidy
```

    ## # A tibble: 18 × 4
    ##    movie           race   gender words
    ##    <chr>           <chr>  <chr>  <dbl>
    ##  1 fellowship_ring elf    female  1229
    ##  2 fellowship_ring elf    male     971
    ##  3 fellowship_ring hobbit female    14
    ##  4 fellowship_ring hobbit male    3644
    ##  5 fellowship_ring man    female     0
    ##  6 fellowship_ring man    male    1995
    ##  7 two_towers      elf    female   331
    ##  8 two_towers      elf    male     513
    ##  9 two_towers      hobbit female     0
    ## 10 two_towers      hobbit male    2463
    ## 11 two_towers      man    female   401
    ## 12 two_towers      man    male    3589
    ## 13 return_king     elf    female   183
    ## 14 return_king     elf    male     510
    ## 15 return_king     hobbit female     2
    ## 16 return_king     hobbit male    2673
    ## 17 return_king     man    female   268
    ## 18 return_king     man    male    2459

## Joing FAS datasets

Import `litters` dataset.

``` r
litter_df = 
  read_csv("./data_import_examples/FAS_litters.csv",na = c("NA", ".", "")) |>
  janitor::clean_names() |>
  separate(group, into = c("dose", "day_of_tx"), sep = 3) |>
  relocate(litter_number) |>
  mutate(
    wt_gain = gd18_weight - gd0_weight,
    dose = str_to_lower(dose))
```

    ## Rows: 49 Columns: 8
    ## ── Column specification ────────────────────────────────────────────────────────
    ## Delimiter: ","
    ## chr (2): Group, Litter Number
    ## dbl (6): GD0 weight, GD18 weight, GD of Birth, Pups born alive, Pups dead @ ...
    ## 
    ## ℹ Use `spec()` to retrieve the full column specification for this data.
    ## ℹ Specify the column types or set `show_col_types = FALSE` to quiet this message.

Import `pups` next:

``` r
pups_df = 
  read_csv("./data_import_examples/FAS_pups.csv", na = c("NA", "."), skip = 3) |>
  janitor::clean_names() |>
  mutate(
    sex = 
      case_match(
        sex, 
        1 ~ "male", 
        2 ~ "female"),
    sex = as.factor(sex)) 
```

    ## Rows: 313 Columns: 6
    ## ── Column specification ────────────────────────────────────────────────────────
    ## Delimiter: ","
    ## chr (1): Litter Number
    ## dbl (5): Sex, PD ears, PD eyes, PD pivot, PD walk
    ## 
    ## ℹ Use `spec()` to retrieve the full column specification for this data.
    ## ℹ Specify the column types or set `show_col_types = FALSE` to quiet this message.

Join the datasets!

There are four major ways join dataframes x and y: Inner: keeps data
that appear in both x and y Left: keeps data that appear in x Right:
keeps data that appear in y Full: keeps data that appear in either x or
y

Left joins are the most common, because they add data from a smaller
table y into a larger table x without removing anything from x.

``` r
fas_df = 
  left_join(pups_df, litter_df, by = "litter_number") |>
  select(litter_number, everything())

fas_df
```

    ## # A tibble: 313 × 15
    ##   litter_number sex   pd_ears pd_eyes pd_pivot pd_walk dose  day_of_tx
    ##   <chr>         <fct>   <dbl>   <dbl>    <dbl>   <dbl> <chr> <chr>    
    ## 1 #85           male        4      13        7      11 con   7        
    ## 2 #85           male        4      13        7      12 con   7        
    ## 3 #1/2/95/2     male        5      13        7       9 con   7        
    ## 4 #1/2/95/2     male        5      13        8      10 con   7        
    ## 5 #5/5/3/83/3-3 male        5      13        8      10 con   7        
    ## # ℹ 308 more rows
    ## # ℹ 7 more variables: gd0_weight <dbl>, gd18_weight <dbl>, gd_of_birth <dbl>,
    ## #   pups_born_alive <dbl>, pups_dead_birth <dbl>, pups_survive <dbl>,
    ## #   wt_gain <dbl>
