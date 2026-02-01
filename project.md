Group15! BlueBike Analysis
================
2025-12-10

# Introduction and Motivation

BlueBikes is one of Boston’s largest public bike-share systems, with
stations spread across Boston, Cambridge, Brookline, and nearby
neighborhoods. As frequent users, we’ve all run into the same practical
frustrations: arriving at an empty station when we need a bike, reaching
a full station when trying to dock, or taking longer routes because
nearby stations are oddly spaced. These small problems suggest a bigger
question — is there an underlying pattern to how BlueBikes stations are
placed and used across the city?

In this project, we use one year of BlueBikes trip data and the
corresponding station list to explore how usage varies across time,
location, rider type, and bike type. Our main goals are to understand:  
1. How do station locations (such as population density, proximity to
colleges, and nearby commercial areas) relate to BlueBikes stations
running empty or full, and what can patterns in start/end-times tell us
about when and where congestion occurs?  
2. How seasonal trends affect the system, such as expected winter drops
or summer peaks.  
3. How distance and trip duration relate, especially because BlueBikes
charges overtime fees after a threshold; stations with many
long-duration trips may indicate gaps where an intermediate station
could reduce overage charges.  
4. Whether there is a correlation between membership type (member
vs. casual) and bike type (classic vs. electric).

Beyond understanding the system, our findings may help identify places
where additional stations, better spacing, or policy changes could
improve the user experience. For instance, if we find strong patterns
around universities such as Boston University, it may support advocacy
for student-focused partnerships or discounted plans. Similarly,
evidence of long-haul trips caused by poor station spacing could support
proposals for intermediate stations or adjusted overtime policies.

To investigate these questions, we collect and clean BlueBikes system
data from the Boston open data portal, perform exploratory analyses on
temporal and spatial trends, and build statistical models to quantify
relationships between demand, location, season, and station
characteristics. Our goal is to produce both practical insights and a
foundation for future planning recommendations.

# Data Collection and Cleaning

We obtained one full year of trip data from the Bluebikes “System Data”
portal (<https://bluebikes.com/system-data>), which publishes monthly
ride history files as CSVs
(<https://s3.amazonaws.com/hubway-data/index.html>). For each month from
October 2024 through October 2025, we downloaded the corresponding
\*-bluebikes-tripdata.csv files converted them to Parquet format as
shown above to allow us a more efficient analysis for large datasets. We
also looked at a list of the BlueBikes stations found on the same page.

For a general data cleaning, we removed all NA values. We also removed
ride IDs since they did not matter in our research. Any other additional
data mutating will be tailored specifically for the question we are
answering.

    ## Rows: 597 Columns: 8
    ## ── Column specification ────────────────────────────────────────────────────────
    ## Delimiter: ","
    ## chr (5): Number, NAME, Seasonal Status, Municipality, Station ID (to match t...
    ## dbl (3): Lat, Long, Total Docks
    ## 
    ## ℹ Use `spec()` to retrieve the full column specification for this data.
    ## ℹ Specify the column types or set `show_col_types = FALSE` to quiet this message.

    ## FileSystemDataset with 12 Parquet files
    ## 13 columns
    ## ride_id: string
    ## rideable_type: string
    ## started_at: timestamp[us, tz=UTC]
    ## ended_at: timestamp[us, tz=UTC]
    ## start_station_name: string
    ## start_station_id: string
    ## end_station_name: string
    ## end_station_id: string
    ## start_lat: double
    ## start_lng: double
    ## end_lat: double
    ## end_lng: double
    ## member_casual: string
    ## 
    ## See $metadata for additional Schema metadata

    ## FileSystemDataset (query)
    ## rideable_type: string
    ## started_at: timestamp[us, tz=UTC]
    ## ended_at: timestamp[us, tz=UTC]
    ## start_station_name: string
    ## start_station_id: string
    ## end_station_name: string
    ## end_station_id: string
    ## start_lat: double
    ## start_lng: double
    ## end_lat: double
    ## end_lng: double
    ## member_casual: string
    ## 
    ## * Filter: (((invert(is_null(start_station_id, {nan_is_null=true})) and invert(is_null(end_station_id, {nan_is_null=true}))) and invert(is_null(started_at, {nan_is_null=true}))) and invert(is_null(ended_at, {nan_is_null=true})))
    ## See $.data for the source Arrow object

# Exploratory Data Analysis

## Question 1

*How do station locations (such as population density, proximity to
colleges, and nearby commercial areas) relate to BlueBikes stations
running empty or full, and what can patterns in start/end-times tell us
about when and where congestion occurs?*

    ## # A tibble: 4,669,660 × 27
    ##    ride_id          rideable_type started_at          ended_at           
    ##    <chr>            <chr>         <dttm>              <dttm>             
    ##  1 41D45A0F4484EE42 electric_bike 2024-11-09 15:04:32 2024-11-09 15:12:26
    ##  2 CC44E57ADF2C19D2 classic_bike  2024-11-01 14:34:44 2024-11-01 14:44:56
    ##  3 3303DF4B71FEE65F electric_bike 2024-11-23 01:15:10 2024-11-23 01:27:39
    ##  4 FBF84AEA696288B2 classic_bike  2024-11-05 22:29:11 2024-11-05 22:48:35
    ##  5 6B14D741EB5650EB electric_bike 2024-11-06 16:39:56 2024-11-06 16:48:43
    ##  6 93BB0BF1F6CEFE1C classic_bike  2024-11-08 09:50:12 2024-11-08 10:03:20
    ##  7 0CD1D4AA59211457 classic_bike  2024-11-11 08:53:21 2024-11-11 09:07:00
    ##  8 63431AFEBBE64D86 classic_bike  2024-11-13 10:25:15 2024-11-13 10:43:11
    ##  9 4EAC9AB0514EC651 classic_bike  2024-11-04 09:17:21 2024-11-04 09:34:25
    ## 10 099D504F3BFC4E18 classic_bike  2024-11-20 09:28:12 2024-11-20 09:43:34
    ## # ℹ 4,669,650 more rows
    ## # ℹ 23 more variables: start_station_name <chr>, start_station_id <chr>,
    ## #   end_station_name <chr>, end_station_id <chr>, start_lat <dbl>,
    ## #   start_lng <dbl>, end_lat <dbl>, end_lng <dbl>, member_casual <chr>,
    ## #   NAME.x <chr>, Lat.x <dbl>, Long.x <dbl>, `Seasonal Status.x` <chr>,
    ## #   start_municipality <chr>, `Total Docks.x` <dbl>,
    ## #   `Station ID (to match to historic system data).x` <chr>, NAME.y <chr>, …

Here we join each trip’s start and end stations with their
municipalities and combine all files into one master dataset. The result
is a single data frame where every ride includes geographic information
for both its origin and destination.

![](project_files/figure-gfm/q1-2_Blue%20Bikes%20in%20Boston-1.png)<!-- -->

![](project_files/figure-gfm/q1-3%20Comparison%20of%20start%20and%20end%20station%20for%201%20year-1.png)<!-- -->

The spatial plot shows that arrival points generally cover larger areas
of intensity compared to departure points. This indicates that many
stations receive more trips ending there than starting, suggesting that
certain neighborhoods act as trip destinations or “sinks” within the
Bluebikes network. These areas likely correspond to workplaces, transit
hubs, universities, or commercial districts where riders commonly finish
their trips.

However, higher arrival intensities should not be interpreted as a
direct signal that these stations need more docks or additional
infrastructure. Bluebikes routinely rebalances bikes across the network,
which can artificially reduce or amplify the observed differences
between arrivals and departures. Daily commuting patterns also
contribute, with more arrivals in central employment zones and more
departures in residential neighborhoods.

The strong arrival intensities may also reflect riders coming from
nearby municipalities such as Brookline and Cambridge-major residential
areas where many students and professionals begin their trips before
commuting into Boston’s job centers.

Therefore, while the visual contrast between arrivals and departures
helps illuminate directional flow patterns in the city, it should be
viewed as a preliminary indicator rather than a standalone measure of
station demand or capacity issues.

    ## Coordinate Reference System:
    ##   User input: WGS 84 
    ##   wkt:
    ## GEOGCRS["WGS 84",
    ##     DATUM["World Geodetic System 1984",
    ##         ELLIPSOID["WGS 84",6378137,298.257223563,
    ##             LENGTHUNIT["metre",1]]],
    ##     PRIMEM["Greenwich",0,
    ##         ANGLEUNIT["degree",0.0174532925199433]],
    ##     CS[ellipsoidal,2],
    ##         AXIS["latitude",north,
    ##             ORDER[1],
    ##             ANGLEUNIT["degree",0.0174532925199433]],
    ##         AXIS["longitude",east,
    ##             ORDER[2],
    ##             ANGLEUNIT["degree",0.0174532925199433]],
    ##     ID["EPSG",4326]]

    ## Coordinate Reference System:
    ##   User input: NAD83 / Massachusetts Mainland 
    ##   wkt:
    ## PROJCRS["NAD83 / Massachusetts Mainland",
    ##     BASEGEOGCRS["NAD83",
    ##         DATUM["North American Datum 1983",
    ##             ELLIPSOID["GRS 1980",6378137,298.257222101,
    ##                 LENGTHUNIT["metre",1]]],
    ##         PRIMEM["Greenwich",0,
    ##             ANGLEUNIT["degree",0.0174532925199433]],
    ##         ID["EPSG",4269]],
    ##     CONVERSION["SPCS83 Massachusetts Mainland zone (meter)",
    ##         METHOD["Lambert Conic Conformal (2SP)",
    ##             ID["EPSG",9802]],
    ##         PARAMETER["Latitude of false origin",41,
    ##             ANGLEUNIT["degree",0.0174532925199433],
    ##             ID["EPSG",8821]],
    ##         PARAMETER["Longitude of false origin",-71.5,
    ##             ANGLEUNIT["degree",0.0174532925199433],
    ##             ID["EPSG",8822]],
    ##         PARAMETER["Latitude of 1st standard parallel",42.6833333333333,
    ##             ANGLEUNIT["degree",0.0174532925199433],
    ##             ID["EPSG",8823]],
    ##         PARAMETER["Latitude of 2nd standard parallel",41.7166666666667,
    ##             ANGLEUNIT["degree",0.0174532925199433],
    ##             ID["EPSG",8824]],
    ##         PARAMETER["Easting at false origin",200000,
    ##             LENGTHUNIT["metre",1],
    ##             ID["EPSG",8826]],
    ##         PARAMETER["Northing at false origin",750000,
    ##             LENGTHUNIT["metre",1],
    ##             ID["EPSG",8827]]],
    ##     CS[Cartesian,2],
    ##         AXIS["easting (X)",east,
    ##             ORDER[1],
    ##             LENGTHUNIT["metre",1]],
    ##         AXIS["northing (Y)",north,
    ##             ORDER[2],
    ##             LENGTHUNIT["metre",1]],
    ##     USAGE[
    ##         SCOPE["Engineering survey, topographic mapping."],
    ##         AREA["United States (USA) - Massachusetts onshore - counties of Barnstable; Berkshire; Bristol; Essex; Franklin; Hampden; Hampshire; Middlesex; Norfolk; Plymouth; Suffolk; Worcester."],
    ##         BBOX[41.46,-73.5,42.89,-69.86]],
    ##     ID["EPSG",26986]]

![](project_files/figure-gfm/q1-4%20Implementing%20Bike%20Lane%20to%20Bike%20Station-1.png)<!-- -->

To understand the spatial distribution of Bluebikes stations across
Boston, we overlay the station locations onto the map of the city’s
recognized bicycle lane facilities. From this plot, we can clearly see
that most stations are situated in close proximity to existing bike
lanes, which aligns with the goal of improving accessibility and safety
for riders. However, a few stations appear to fall outside the immediate
bike-lane network. This could indicate streets that are still
bike-friendly but not formally designated as bike lanes, or areas where
riders commonly travel despite lacking dedicated infrastructure.

When comparing this Bluebikes station map to the earlier departure-point
plot, some inconsistencies in station positions and counts become
noticeable. These discrepancies may be due to data limitations or
updates; such as stations being temporarily closed, relocated, or
decommissioned by Bluebikes due to low usage or strategic restructuring
during the study year. It’s also possible that older station records
remain in the dataset even though the physical docks have been removed.

Overall, this overlay helps assess how well the current station network
aligns with the city’s bike-lane system and highlights areas where
infrastructure updates, data corrections, or additional stations may be
worth exploring.

![](project_files/figure-gfm/q1-5%20Bivariance%20Plot-1.png)<!-- -->

The bivariate map is a useful tool for visualizing the relationship
between two variables-in this case, the bike-lane length within each
Boston subdistrict and the total number of Bluebikes trips originating
or ending there. While this method helps reveal spatial patterns, it is
also somewhat imprecise, since lane length alone does not fully capture
the quality, connectivity, or actual usability of the bike network.
Longer lane segments may include low-traffic side streets, disconnected
paths, or lanes that riders do not frequently choose.

Despite these limitations, the bivariate map in this study shows a clear
pattern: subdistricts with longer bike-lane networks tend to have higher
Bluebikes trip volumes. This suggests that improved cycling
infrastructure is associated with greater system usage, supporting the
idea that accessibility and safety encourage more riders.

However, it is important to account for geographic differences. Some
subdistricts naturally require more lane length because they cover
larger land areas, include more major roads, or have unique physical or
urban constraints. These structural characteristics influence both how
much bike infrastructure can be built and how many trips the area can
support.

Overall, while the bivariate map provides a helpful preliminary
visualization of the relationship between lane length and trip
frequency, it should be complemented with more precise modeling; such as
spatial regression-to fully understand how bike infrastructure
influences Bluebikes usage.

## Question 2

*How seasonal trends affect the system, such as expected winter drops or
summer peaks*

![](project_files/figure-gfm/q2-1-1.png)<!-- -->

The bar plot shows strong seasonal variation in BlueBikes usage over the
year. Usage is lowest in the winter months (December–Feb), increases
steadily through the spring, and stays high throughout the summer
(June–August). This pattern is expected because biking is more appealing
and practical in warmer weather.

Trips peak sharply in September and October, which likely reflects a
combination of:

- returning university students (BU, Northeastern, MIT, Harvard)

- good weather (not too hot, not too cold)

- commuter activity increasing after summer vacations

After October, trip counts drop again moving into November and December
as temperatures fall.

    ## [1] "Number"                                       
    ## [2] "NAME"                                         
    ## [3] "Lat"                                          
    ## [4] "Long"                                         
    ## [5] "Seasonal Status"                              
    ## [6] "Municipality"                                 
    ## [7] "Total Docks"                                  
    ## [8] "Station ID (to match to historic system data)"

    ##  [1] "rideable_type"      "started_at"         "ended_at"          
    ##  [4] "start_station_name" "start_station_id"   "end_station_name"  
    ##  [7] "end_station_id"     "start_lat"          "start_lng"         
    ## [10] "end_lat"            "end_lng"            "member_casual"     
    ## [13] "trip_duration_min"  "trip_date"          "trip_hour"         
    ## [16] "trip_month"         "trip_wday"

Just checking our column names and joining station data to get
municipality info for each trip’s start station.

``` r
p_all <- question2 %>%
  count(trip_month) %>%
  ggplot(aes(x = trip_month, y = n)) +
  geom_col(fill = "steelblue") +
  labs(
    title = "Total BlueBikes Trips by Month",
    x = "Month", y = "Number of Trips"
  )

p_boston <- question2 %>%
  filter(!is.na(start_municipality),
         start_municipality == "Boston") %>%
  count(trip_month) %>%
  ggplot(aes(x = trip_month, y = n)) +
  geom_col(fill = "steelblue") +
  labs(
    title = "BlueBikes Trips from Boston Stations by Month",
    x = "Month", y = "Number of Trips"
  )

p_all | p_boston
```

![](project_files/figure-gfm/unnamed-chunk-1-1.png)<!-- -->

Placing these plots side by side shows that trips starting at Boston
stations follow the same seasonal pattern as overall system usage, with
low activity in winter and peaks in late summer and early fall. While
total trip counts are lower for Boston-only stations, the timing of
increases and declines is closely aligned, suggesting that city stations
primarily drive the system’s seasonal trends.

![](project_files/figure-gfm/q2_lineplot-1.png)<!-- -->![](project_files/figure-gfm/q2_lineplot-2.png)<!-- -->

Members make more trips than casual riders in every month, though both
groups follow similar seasonal patterns. Interesting to note however, is
that September sees an increase in trips among member users, while
casual users peak in the summer months of June and July. This suggests
that members may be more influenced by the academic calendar, while
casual riders are more driven by leisure and tourism during summer;
casual users see a dip in trip being made starting in September. At any
given month, member users constantly make more trips than casual users
by a large margin.

Across seasons, trips peak during daytime hours, with the strongest and
longest activity windows occurring in summer and fall. Also important to
note is that trips across all 4 seasons seem to peak at around 8am, and
then again 5pm. The morning peak likely reflects commuting trips, such
as riders traveling to work, school, or transit hubs. The larger
afternoon/early-evening peak is consistent with return commutes; the
even greater average number of trips than morning can attributed to
people riding bikes for social or recreational travel after
work/school/business throughout the day.

## Question 3

*How distance and trip duration relate, especially because BlueBikes
charges overtime fees after a threshold; stations with many
long-duration trips may indicate gaps where an intermediate station
could reduce overage charges.*

    ## 
    ## Attaching package: 'scales'

    ## The following object is masked from 'package:purrr':
    ## 
    ##     discard

    ## The following object is masked from 'package:readr':
    ## 
    ##     col_factor

    ## function (x, y, ...) 
    ## UseMethod("plot")
    ## <bytecode: 0x000001f24004b1c8>
    ## <environment: namespace:base>

![](project_files/figure-gfm/q3_overtimeplot-1.png)<!-- -->

Between 1 and 3 km, the overage rate is minimal (\<5%), suggesting that
most trips within this distance are completed within the time limit.
However, starting from 3 km, the overage rate begins to rise noticeably,
reaching around 15% at 4-5 km and climbing to nearly 30% for trips
between 7-8 km. This indicates that longer trips are more likely to
exceed the time limit, leading to additional fees.  
We should also point out trips below 1 km have a slightly higher overage
rate than 1-3 km trips. This could be due to short-distance trips where
riders take longer routes or due to station capacity limits, leading to
exceeding the time limit despite the short distance.

    ## # A tibble: 20 × 5
    ##    start_station_name                    end_station_name count avg_mins  avg_km
    ##    <chr>                                 <chr>            <int>    <dbl>   <dbl>
    ##  1 Mugar Way at Beacon St                Mugar Way at Be…  1102     64.7 5.55e-5
    ##  2 Charles Circle - Charles St at Cambr… Charles Circle …   794     67.6 2.76e-4
    ##  3 Eliot Circle Revere Beach             Eliot Circle Re…   694     67.3 7.37e-5
    ##  4 Pope John Paul II Park - Neponset Tr… Pope John Paul …   680     59.9 2.52e-4
    ##  5 Murphy Skating Rink - 1880 Day Blvd   Murphy Skating …   679     66.1 1.30e-4
    ##  6 Herter Park                           Herter Park        660     64.7 4.31e-5
    ##  7 Revere Beach - Markey Footbridge      Revere Beach - …   618     65.2 7.66e-5
    ##  8 Harvard University River Houses at D… Harvard Univers…   504     63.6 9.64e-5
    ##  9 Beacon St at Charles St               Beacon St at Ch…   491     85.3 7.04e-5
    ## 10 Christian Science Plaza - Massachuse… Christian Scien…   398     79.7 1.02e-3
    ## 11 Farragut Rd at E. 6th St              Farragut Rd at …   394     62.7 4.76e-4
    ## 12 Beacon St at Massachusetts Ave        Beacon St at Ma…   390     63.6 1.99e-4
    ## 13 Harvard Square at Mass Ave/ Dunster   Harvard Square …   386     76.1 1.19e-4
    ## 14 Northern Strand at Salem Street       Northern Strand…   369     72.3 2.75e-5
    ## 15 Lewis Wharf at Atlantic Ave           Lewis Wharf at …   350     87.2 2.75e-4
    ## 16 7 Acre Park                           7 Acre Park        339     69.6 8.76e-4
    ## 17 Boylston St at Arlington St           Boylston St at …   333     66.1 3.07e-5
    ## 18 Massachusetts Ave at Boylston St.     Massachusetts A…   322     71.9 2.80e-4
    ## 19 MIT at Mass Ave / Amherst St          MIT at Mass Ave…   319     70.8 1.97e-4
    ## 20 Airport T Stop - Bremen St at Brooks… Airport T Stop …   314     65.1 8.78e-5

    ## # A tibble: 20 × 5
    ##    start_station_name                     end_station_name count avg_mins avg_km
    ##    <chr>                                  <chr>            <int>    <dbl>  <dbl>
    ##  1 Beacon St at Charles St                Mugar Way at Be…   198     69.8  0.255
    ##  2 Mugar Way at Beacon St                 Charles Circle …   172     63.6  0.601
    ##  3 Mugar Way at Beacon St                 Beacon St at Ch…   160     70.5  0.255
    ##  4 MIT at Mass Ave / Amherst St           Harvard Square …   140     56.2  2.67 
    ##  5 Christian Science Plaza - Massachuset… Massachusetts A…   127     59.5  0.463
    ##  6 Mugar Way at Beacon St                 Beacon St at Ma…   109     61.4  1.49 
    ##  7 Charles Circle - Charles St at Cambri… Mugar Way at Be…   107     78.1  0.601
    ##  8 Harvard Square at Mass Ave/ Dunster    MIT at Mass Ave…   106     93.5  2.67 
    ##  9 Beacon St at Charles St                Charles Circle …   105     91.4  0.539
    ## 10 Harvard University River Houses at De… MIT at Mass Ave…    97     43.8  2.29 
    ## 11 Charles Circle - Charles St at Cambri… Beacon St at Ma…    96     62.3  1.89 
    ## 12 Farragut Rd at E. 6th St               Murphy Skating …    91     59.8  0.353
    ## 13 Beacon St at Charles St                Boylston St at …    89     72.6  0.441
    ## 14 Murphy Skating Rink - 1880 Day Blvd    Farragut Rd at …    81     78.2  0.354
    ## 15 Boylston St at Arlington St            Beacon St at Ch…    78     65.1  0.440
    ## 16 MIT at Mass Ave / Amherst St           Mugar Way at Be…    77     74.8  1.71 
    ## 17 Boylston St at Arlington St            Mugar Way at Be…    75     58.7  0.457
    ## 18 Boylston St at Massachusetts Ave       Massachusetts A…    74     59.9  0.102
    ## 19 Railroad Lot and Minuteman Bikeway     Mill St. at Min…    73    113.   0.444
    ## 20 Valenti Way at Haverhill St            Canal St at Cau…    73    134.   0.131

![](project_files/figure-gfm/q3_truegap-1.png)<!-- -->

Looking at the plot, we see that out of the 15 problem routes, 10 of
them are trips between Harvard University locations and MIT. This
suggests that the single most inefficient commuter corridor in the
entire BlueBikes network (after filtering) is the stretch of
Massachusetts Ave connecting these two universities. The expected trip
duration all hover between 10 and 15 minutes, suggesting that the
physical distance is short and direct (around 2-3km). However, the
actual durations hover around 45 to 55 minutes, which are about 3x
longer than what the physical distance implies.

This might be due to a docking availability problem where students ride
to Harvard/MIT stations and find no available docks, forcing them to
ride further to find an open station. A possible solution for this
“inefficiency” could be to add more docks at these high-traffic
stations.

Users can request an additional 15 minutes at the kiosk of stations if
the docks are full, which stops overcharge fees from accruing. However,
this only helps if users are aware of this option and willing to take
the extra time to find an open dock.

One interesting outlier was from Mugar Way to Silber Way which is along
the Charles River Esplanade. This delay (45 mins vs 12 expected) could
suggest that users recreationally riding along the river path, rather
than commuting. Another outlier is from Bike Town to Murphy Skating
Rink, which could also be a recreational beach riding trip rather than a
commute.

![](project_files/figure-gfm/q3_isolation-1.png)<!-- -->

The graph above shows the 20 most isolated BlueBike stations based on
median trip duration after departing. The longer the median trip time,
the more “isolated” the station is considered. The median trip duration
for a bike leaving Auburndale and Northern Strand are 30+ minutes for
6.3 km and 5.8 km, respectfully. This suggests that these stations are
poorly connected to the rest of the BlueBike network, requiring long
rides to reach other stations. A suggestion for BlueBikes would be to
add intermediate stations roughly 2-3 km away from these outlier
stations, so people have the choice of taking shorter trips rather than
long rides.

# Modeling and Inference

## Question 1

    ## 
    ## Call:
    ## lm(formula = total_trips ~ lane_length_m, data = boston_metrics)
    ## 
    ## Residuals:
    ##     Min      1Q  Median      3Q     Max 
    ## -229893 -125270 -101448   55611  756824 
    ## 
    ## Coefficients:
    ##                Estimate Std. Error t value Pr(>|t|)    
    ## (Intercept)   1.458e+05  1.323e+04   11.02   <2e-16 ***
    ## lane_length_m 9.592e+00  4.223e-01   22.71   <2e-16 ***
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
    ## 
    ## Residual standard error: 245100 on 695 degrees of freedom
    ## Multiple R-squared:  0.4261, Adjusted R-squared:  0.4252 
    ## F-statistic:   516 on 1 and 695 DF,  p-value: < 2.2e-16

![](project_files/figure-gfm/q1%20-%20Modelling%201-1.png)<!-- -->

    ## 
    ##  Moran I test under randomisation
    ## 
    ## data:  boston_metrics$total_trips  
    ## weights: lw  
    ## n reduced by no-neighbour observations  
    ## 
    ## Moran I statistic standard deviate = 26.113, p-value < 2.2e-16
    ## alternative hypothesis: greater
    ## sample estimates:
    ## Moran I statistic       Expectation          Variance 
    ##      0.6839825844     -0.0014388489      0.0006889684

    ## 
    ##  Moran I test under randomisation
    ## 
    ## data:  boston_metrics$lane_length_m  
    ## weights: lw  
    ## n reduced by no-neighbour observations  
    ## 
    ## Moran I statistic standard deviate = 35.594, p-value < 2.2e-16
    ## alternative hypothesis: greater
    ## sample estimates:
    ## Moran I statistic       Expectation          Variance 
    ##      0.9347775489     -0.0014388489      0.0006918186

    ## 
    ## Call:lagsarlm(formula = total_trips ~ lane_length_m, data = boston_metrics, 
    ##     listw = lw, zero.policy = TRUE)
    ## 
    ## Residuals:
    ##     Min      1Q  Median      3Q     Max 
    ## -601801  -87497  -10611   45264  997473 
    ## 
    ## Type: lag 
    ## Regions with no neighbours included:
    ##  595 
    ## Coefficients: (numerical Hessian approximate standard errors) 
    ##                 Estimate Std. Error z value Pr(>|z|)
    ## (Intercept)   7.6821e+03 1.3433e+04  0.5719   0.5674
    ## lane_length_m 4.3602e+00 3.7208e-01 11.7184   <2e-16
    ## 
    ## Rho: 0.69935, LR test value: 401.27, p-value: < 2.22e-16
    ## Approximate (numerical Hessian) standard error: 0.028574
    ##     z-value: 24.475, p-value: < 2.22e-16
    ## Wald statistic: 599.02, p-value: < 2.22e-16
    ## 
    ## Log likelihood: -9436.795 for lag model
    ## ML residual variance (sigma squared): 3.0332e+10, (sigma: 174160)
    ## Number of observations: 697 
    ## Number of parameters estimated: 4 
    ## AIC: 18882, (AIC for lm: 19281)

The statistical results indicate a strong and significant relationship
between bicycle lane length and the number of BlueBike trips across
Boston subdistricts. The OLS regression shows that lane length is a
positive predictor of ridership (Estimated at 9.59, p-value \< 2e-16),
meaning that for every additional meter of bicycle lane, the model
predicts roughly 9-10 more annual trips. The model explains about 42% of
the variation in trip counts (R-squared estimated at 0.43), suggesting a
moderate but meaningful association. However, the very high and
significant Moran’s I values for both total trips (I estimated at 0.68)
and lane length (I estimated at 0.93) indicate strong spatial
autocorrelation, meaning that subdistricts with high values tend to
cluster together geographically. This violates the independence
assumption of OLS and suggests that spatial processes influence
ridership patterns.

After accounting for spatial dependence using a spatial lag model, the
relationship between lane length and trips remains strongly significant,
though the effect size decreases (Estimated at 4.36). The spatial lag
coefficient (rho is estimated at 0.70, p-value \< 2.2e-16) indicates
that nearby subdistricts have a substantial influence on each other’s
ridership levels. This means that areas adjacent to high-usage zones are
likely to also have higher trip volumes, independent of lane length
alone. The spatial model provides a better fit (AIC 18882 vs. 19281 for
OLS), confirming that spatial effects are essential to explaining trip
distribution. Overall, the analysis shows that longer bicycle lane
networks are associated with higher BlueBike usage, but ridership
patterns are also strongly shaped by spatial clustering, neighboring
infrastructure, and broader geographic context.

The linear regression model shows a strong and statistically significant
relationship between bike lane length and total BlueBike trips. The
slope of 9.59 (p \< 2e-16) indicates that, on average, each additional
meter of bike lane is associated with roughly 9-10 more trips. The model
explains about 42.6% of the variation in trip counts (R-squared =
0.426), suggesting that lane length accounts for a meaningful portion of
ridership but that other factors; such as population density, station
placement, and spatial clustering; also play important roles. The
residual standard error is relatively large, indicating considerable
variability around the fitted line, so while the positive relationship
is clear, the linear fit does not capture all the complexity in the
data. Overall, the model is valid and interpretable, but additional
modeling or transformations could improve fit and account for
nonlinearity or spatial effects.

    ## 
    ## Call:
    ## lm(formula = total_trips ~ log_lane_length, data = boston_metrics)
    ## 
    ## Residuals:
    ##     Min      1Q  Median      3Q     Max 
    ## -338853  -85754  -37606    1746  710093 
    ## 
    ## Coefficients:
    ##                 Estimate Std. Error t value Pr(>|t|)    
    ## (Intercept)     -1124132      65113  -17.26   <2e-16 ***
    ## log_lane_length   158675       6892   23.02   <2e-16 ***
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
    ## 
    ## Residual standard error: 243700 on 695 degrees of freedom
    ## Multiple R-squared:  0.4327, Adjusted R-squared:  0.4319 
    ## F-statistic: 530.1 on 1 and 695 DF,  p-value: < 2.2e-16

![](project_files/figure-gfm/q1%20-%20Modelling%202-1.png)<!-- -->

The log-transformed linear model shows a strong, significant positive
relationship between bike lane length and total BlueBike trips. The
slope indicates that proportional increases in lane length lead to
substantial increases in trips, though the intercept is not directly
meaningful. The model explains about 43% of the variation in trips,
suggesting a moderate fit, while the residuals indicate that other
factors; like station placement or neighborhood characteristics; also
influence ridership. Overall, logging lane length helps account for skew
and confirms that longer bike lanes are associated with higher usage.

    ## 
    ## Formula: total_trips ~ a * log_lane^b
    ## 
    ## Parameters:
    ##   Estimate Std. Error t value Pr(>|t|)    
    ## a   3.2088     2.0622   1.556     0.12    
    ## b   5.1243     0.2734  18.743   <2e-16 ***
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
    ## 
    ## Residual standard error: 233700 on 695 degrees of freedom
    ## 
    ## Number of iterations to convergence: 29 
    ## Achieved convergence tolerance: 8.327e-08

![](project_files/figure-gfm/q1%20-%20modelling%203-1.png)<!-- -->![](project_files/figure-gfm/q1%20-%20modelling%203-2.png)<!-- -->
The nonlinear model suggests that BlueBike trips increase in a nonlinear
fashion with bike lane length. The exponent parameter b = 5.12 is highly
significant, indicating a steep increase in trips as lane length grows,
while the scaling factor a=3.21 is not statistically significant. The
residual standard error is slightly lower than in the linear models, and
the model explains a substantial proportion of the variance in trip
counts, indicating a strong fit. Overall, this curve implies that longer
bike lanes are associated with disproportionately higher usage,
reflecting a strong nonlinear effect of infrastructure on ridership.

## Question 3

    ## 
    ## Call:
    ## lm(formula = overtime_rate ~ dist_mid, data = overtime_lm_data)
    ## 
    ## Residuals:
    ##      Min       1Q   Median       3Q      Max 
    ## -0.03648 -0.02469 -0.01813  0.01196  0.06961 
    ## 
    ## Coefficients:
    ##              Estimate Std. Error t value Pr(>|t|)    
    ## (Intercept) -0.055806   0.025125  -2.221   0.0571 .  
    ## dist_mid     0.049245   0.004357  11.302 3.38e-06 ***
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
    ## 
    ## Residual standard error: 0.03958 on 8 degrees of freedom
    ## Multiple R-squared:  0.9411, Adjusted R-squared:  0.9337 
    ## F-statistic: 127.7 on 1 and 8 DF,  p-value: 3.38e-06

    ## `geom_smooth()` using formula = 'y ~ x'

![](project_files/figure-gfm/q3_overtimeplot%20-%20modelling%201-1.png)<!-- -->

The linear model examining the relationship between trip distance and
the proportion of trips incurring overage fees shows a strong positive
association. The slope coefficient for distance (dist_mid) is 0.049,
highly significant (p \< 0.001), indicating that as trip distance
increases by 1 km, the overage rate increases by roughly 4.9 percentage
points. The intercept is slightly negative but marginally insignificant
(p estimated at 0.057). The model explains a very large portion of the
variation in overage rates, with an R-squared of 0.94, suggesting an
excellent fit. Overall, the results indicate that longer trips are
strongly associated with a higher likelihood of exceeding the allowed
time limit.

![](project_files/figure-gfm/q3_overtimeplot%20-%20modelling%202-1.png)<!-- -->![](project_files/figure-gfm/q3_overtimeplot%20-%20modelling%202-2.png)<!-- -->

## Question 4

*Whether there is a correlation between membership type (member
vs. casual) and bike type (classic vs. electric).*

![](project_files/figure-gfm/q4-1.png)<!-- -->

    ## # A tibble: 2 × 3
    ## # Groups:   member_casual [2]
    ##   member_casual classic_bike electric_bike
    ##   <chr>                <int>         <int>
    ## 1 casual             1007147        324557
    ## 2 member             2352160        976395

    ## 
    ##  Pearson's Chi-squared test with Yates' continuity correction
    ## 
    ## data:  contingency_matrix
    ## X-squared = 11639, df = 1, p-value < 2.2e-16

    ##                    X^2 df P(> X^2)
    ## Likelihood Ratio 11840  1        0
    ## Pearson          11640  1        0
    ## 
    ## Phi-Coefficient   : 0.05 
    ## Contingency Coeff.: 0.05 
    ## Cramer's V        : 0.05

The p-value from the chi-squared test is very small (p \< 0.001),
indicating a significant correlation between membership type and bike
type. However, the Cramer’s V value is 0.05 which indicates a weak
association. This means we might have to look at other factors that can
influence bike type choice beyond membership status.

# Conclusion

Overall, our analysis shows that BlueBikes usage patterns across Boston
are highly structured in space and time, with demand concentrating
around dense urban corridors, major institutions, and predictable
commute periods. These recurring patterns suggest that while the system
is generally well-aligned with the city’s travel needs, certain
locations experience sustained pressure.

Although BlueBikes actively rebalances bikes across stations, our
results indicate that rebalancing alone does not always resolve these
pressures. In several high-demand corridors and destination-heavy areas,
trips are consistently longer than expected given their physical
distance, pointing to limitations in dock capacity and network
connectivity rather than temporary bike shortages.

Taken together, these findings suggest that targeted interventions—such
as increasing dock capacity at major destination stations, refining
rebalancing schedules during peak hours, and adding intermediate
stations in poorly connected areas—would be more effective than broad
system-wide changes. Our study demonstrates how combining trip data with
spatial analysis can support focused, data-driven improvements to
bike-share infrastructure.
