Agriculture Emissions Data Wrangling
================
Ben DeMoras
2/8/2022

## Data Needed:

-   N2O emissions (Response variable)

    -   Broken down into synthetic fertilizers, manure applied on soils,
        and N2O emissions. *Manure applied on soils* is what farmers are
        actively applying as fertilizer, whereas *left on soils* is the
        manure deposited by grazing livestock. I only include *applied
        to soils* because you don’t have synthetic-pooping livestock.

    -   Filter down to N2O emissions from synthetic and total manure.

    -   Source: See below.

-   Manure and Synthetic Fertilizer use: (Treatment)

    -   Filter down to tons of synthetic fertilizer and tons manure
        applied

        -   Synthetic: <https://www.fao.org/faostat/en/#data/GY>

        -   Manure (applied to soil):
            <https://www.fao.org/faostat/en/#data/GU>

    -   Filter agricultural cropland area data
        <https://www.fao.org/faostat/en/#data/RL>

    -   Calculate manure/fertilizer use per acre

## Import Data

I am Using Tidyverse packages to more easily import data to R and filter
down what we need. Cheatsheet:
<https://www.rstudio.com/wp-content/uploads/2015/02/data-wrangling-cheatsheet.pdf>

``` r
# install.packages('dplyr')
  library(dplyr)

# 1. Import Data
manure.csv <- read.csv(file="csv-data/Emissions_Manure.csv", header = TRUE)
synth.csv  <- read.csv(file="csv-data/Emissions_Synthetic.csv", header = TRUE)
```

## Tidy Data

NOTE: FAO breaks down fertilizers according to how the emissions end up.
We want to consider these as one source. **We need to be sure we are
combining the right categories!**

``` r
# 2. Define some variables we'll need later on.
manure.use.filter = c(
    "Manure applied to soils (N content)",
    "Manure applied to soils that leaches (N content)",
    "Manure applied to soils that volatilises (N content)")

manure.emit.filter = c(
    "Direct emissions (N2O) (Manure applied)",
    "Indirect emissions (N2O that leaches) (Manure applied)",
    "Indirect emissions (N2O that volatilises) (Manure applied)",
    "Indirect emissions (N2O) (Manure applied)")

synth.use.filter = c(
    "Agricultural Use in nutrients",
    "Nitrogen fertilizer content applied that leaches",
    "Nitrogen fertilizer content applied that volatilises")

synth.emit.filter = c(
    "Direct emissions (N2O) (Synthetic fertilizers)",
    "Indirect emissions (N2O that leaches) (Synthetic fertilizers)",
    "Indirect emissions (N2O that volatilises) (Synthetic fertilizers)",
    "Indirect emissions (N2O) (Synthetic fertilizers)")

# 3. Get only the data we need
manure.use <- manure.csv %>% 
    filter(Element %in% manure.use.filter) %>%
    select(Area, Y2019) %>% 
    # Manure data are broken down by animal so now we need to combine them by country
    group_by(Area) %>% 
    summarize(Kg.ManureN.2019 = sum(Y2019, na.rm = TRUE))

manure.emit <- manure.csv %>% 
    filter(Element %in% manure.emit.filter) %>% 
    select(Area, Y2019) %>%
    group_by(Area) %>% 
    summarize(KT.ManureN2O.2019 = sum(Y2019, na.rm = TRUE))

synth.use <- synth.csv %>%
    filter(Element %in% synth.use.filter) %>% 
    select(Area, Y2019) %>% 
    group_by(Area) %>% 
    summarize(Kg.SynthNutrient.2019 = sum(Y2019, na.rm = TRUE))

synth.emit <- synth.csv %>% 
    filter(Element %in% synth.emit.filter) %>% 
    select(Area, Y2019) %>% 
    group_by(Area) %>% 
    summarize(KT.SynthN20.2019 = sum(Y2019, na.rm = TRUE))

# 4. Display subset of results in console to confirm success
# head(manure.use)
# head(manure.emit)
# head(synth.use)
# head(synth.emit)
```

## Combine the data

Now, we’re ready to combine the data into one dataframe for analysis!

``` r
data.master <- manure.use %>% 
    inner_join(manure.emit, by = "Area") %>% 
    inner_join(synth.use,   by = "Area") %>% 
    inner_join(synth.emit,  by = "Area")

head(data.master, n = 10L)
```

    ## # A tibble: 10 x 5
    ##    Area       Kg.ManureN.2019 KT.ManureN2O.20~ Kg.SynthNutrien~ KT.SynthN20.2019
    ##    <chr>                <dbl>            <dbl>            <dbl>            <dbl>
    ##  1 Afghanist~      316879282.           6.14         132561428.           2.46  
    ##  2 Africa         7406128676.         144.          5772222912.         107.    
    ##  3 Albania         140873997.           2.73          49863363.           0.924 
    ##  4 Algeria         125380154.           2.43          98280000            1.82  
    ##  5 Americas      30591699613.         593.         33818750002.         626.    
    ##  6 Angola          290838889.           5.64          32402553.           0.600 
    ##  7 Annex I c~    47077478320.         912.         45947862910.         851.    
    ##  8 Antigua a~         977231.           0.0185           36743.           0.0006
    ##  9 Argentina       872929652.          16.9         1430765647.          26.5   
    ## 10 Armenia          21842562.           0.423        152368907.           2.82

We still need to exclude some categories (ie, `Annex I countries`,
`Africa`) and figure out if we’ll analyze the whole dataset, or a random
subset.
