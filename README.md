# Overview of Cloned Repositories

In this README document, I have detailed the three news repositories I cloned, and then how I analyzed them using a built-in AI assistant with VS Code.

## 1. Streetsblog Bike Lanes Map

Repository URL: https://github.com/pspauster/bike-lanes-vz0-priority.git

### Questions I asked the chatbot
#### Q1: Where is the data from the map coming from, and how did the creator make it interactive?

A: The map data comes from multiple sources, all loaded directly within the R script:

NYC Bike Routes (2017): Local shapefile from the Archived_nyc_bike_routes_2017 2 folder
NYC Bike Routes (2023): Loaded via API from NYC Open Data
Vision Zero Priority Districts: Loaded from NYC Open Data
Community Boards: Loaded from NYC Open Data
Cyclist Crashes: CSV data from NYC Open Data crashes database

How Interactivity Was Created
The creator made the map interactive using R's leaflet package, which creates web-based interactive maps using the Leaflet JavaScript library. Key interactive features:

Clickable Polygons: The community board polygons have popups that display when clicked (lines 175-183), showing:

Community board name
Number of people injured since 2017
Number of people killed since 2017
Layered Data: Multiple layers (polygons for boundaries, polylines for bike routes) allow visual comparison

Custom Styling: CSS embedded in the HTML controls the appearance and responsive design

Leaflet Controls: Zoom controls, legend, and a custom title control

The final map is exported as a standalone HTML file (map.html) using mapview::mapshot(), which embeds all the geographic data as GeoJSON directly into the HTML file's JavaScript, making it fully self-contained and interactive without needing a server.

#### Q2: Explain what R script is, and how it is used here to combine multiple data sets?

A: R is a programming language designed for statistical computing and data analysis. An R script (like analysis.R) is a text file containing R code that performs data analysis tasks. R is particularly popular in journalism, academia, and data science for:

Loading and cleaning data
Statistical analysis
Creating visualizations
Working with geographic/spatial data

This script uses spatial joins and data merging to combine five different datasets. Here's the process:

1. Loading the Data
Load 2017 bike lanes from local files
bike_lanes_17 <- read_sf("Archived_nyc_bike_routes_2017 2")

Load 2023 bike lanes from NYC Open Data API
bike_lanes_23 <- read_sf("https://data.cityofnewyork.us/resource/s5uu-3ajy.geojson")

Load Vision Zero priority districts
priority_districts <- read_sf("https://data.cityofnewyork.us/resource/fxnz-5gsp.geojson")

Load community boards
community_boards <- read_sf("https://data.cityofnewyork.us/resource/jp9i-3b7y.geojson")

Load crash data
crashes_cyclists <- read_csv("https://data.cityofnewyork.us/resource/h9gi-nx95.csv")

2. Spatial Joining (Combining Geographic Data)
The script uses st_intersection() to find where datasets overlap geographically:

Lines 75-94: Creates cb_stats by intersecting bike lanes with priority districts:
Find which 2017 bike lanes are in priority districts
st_intersection(bike_lanes_17, priority_districts) %>% 
  group_by(boro_cd_cod) %>% 
  summarize(route_length_17 = sum(length, na.rm = T)/5280)

This combines bike route geometry with community board boundaries to calculate total route length per district.

Lines 37-41: Links crash locations to community boards:
crashes_cyclists_sf <- crashes_cyclists %>% 
  st_as_sf(coords = c("longitude", "latitude")) %>%  # Convert to spatial data
  st_intersection(community_boards)  # Find which CB each crash is in

3. Table Joining (Combining Statistics)
The script uses left_join() to merge summary statistics:

Lines 85-98: Combines three separate calculations into one table:
cb_stats <- [2017 bike lane lengths] %>%
  left_join([2023 bike lane lengths], by = "boro_cd_cod") %>%
  left_join([crash statistics], by = "boro_cd_cod") %>%
  mutate(miles_new = route_length_23 - route_length_17)  # Calculate new lanes

This creates a comprehensive dataset with:

2017 bike lane miles per community board
2023 bike lane miles per community board
Injury and death counts per community board
Calculated field for new bike lane miles

4. The Final Result
The combined cb_stats dataset contains all the information needed for the interactive map: geometry (polygon boundaries), bike lane statistics, and crash data—all linked by community board ID (boro_cd_cod). This single merged dataset powers the map's popups and visualization layers.

Key R Techniques Used:

st_intersection() - spatial join (geographic overlap)
left_join() - table join (matching by ID)
group_by() + summarize() - aggregate statistics
mutate() - create new calculated fields
Pipe operator %>% - chain operations together

#### Q3: Explain to me what HTMLwidgets and viewer pane is?

A: HTMLWidgets is an R framework that creates interactive JavaScript visualizations from R code. It acts as a bridge between R and JavaScript libraries.

How it works:

You write R code (like leaflet() in analysis.R)
HTMLWidgets automatically generates HTML/JavaScript
The output can be viewed interactively or saved as a standalone HTML file
In this project, the leaflet package uses HTMLWidgets to convert R commands like this:

leaflet() %>% 
  addProviderTiles("CartoDB.Positron") %>% 
  addPolygons(data = cb_stats)

Into the interactive JavaScript map you see in map.html.

Looking at lines 7-10 in the HTML file, you can see HTMLWidgets initializing:

// If window.HTMLWidgets is already defined, then use it; otherwise create a
// new object. This allows preceding code to set options that affect the
// initialization process (though none currently exist).
window.HTMLWidgets = window.HTMLWidgets || {};

The Viewer Pane is a panel in RStudio (the IDE for R) that displays HTML outputs like plots, maps, and interactive widgets.

Lines 13-14 check if the map is running in the viewer pane:

var viewerMode = window.HTMLWidgets.viewerMode =
    /\bviewer_pane=1\b/.test(window.location);

When you run the R script in RStudio:

routes_map executes and creates the leaflet map
RStudio automatically displays it in the Viewer Pane (like a mini browser inside RStudio)
You can interact with the map immediately to check if it works
If satisfied, you export it with mapview::mapshot(routes_map, url = "map.html")
The Viewer Pane vs. Standalone HTML:

Viewer Pane: Quick preview inside RStudio during development
Standalone HTML: The exported map.html file that works in any web browser without RStudio
This workflow lets data journalists prototype interactive visualizations in R, preview them instantly, and then export production-ready HTML files for publication.

#### Q4: If I wanted to take this code and reformat the map, so that Community Board boundaries are "grayed out" until a mouse hovers over them, would that be a fairly simple fix?

A: Yes, that's a fairly simple fix! Leaflet has built-in hover functionality called highlightOptions.

You would modify the R script (analysis.R), not the HTML file directly. The relevant section is lines 175-183:

addPolygons(data = cb_stats,
            color = "black",
            fill = "white",
            fillOpacity = 0,
            stroke = TRUE,
            weight = 1,
            popup = ~paste0(boro_cd_cod,
                            "<br><b>Since 2017</b><br>",
                            total_injuries, " people injured <br>",
                            total_deaths, " people killed"
                            ))

Modified code with hover effect:

addPolygons(data = cb_stats,
            color = "#CCCCCC",           # Gray borders initially
            fillColor = "white",
            fillOpacity = 0.1,           # Subtle fill
            stroke = TRUE,
            weight = 1,
            highlightOptions = highlightOptions(
              color = "black",           # Black border on hover
              weight = 2.5,              # Thicker border on hover
              fillOpacity = 0.5,         # More visible fill on hover
              bringToFront = TRUE        # Bring to front when hovering
            ),
            popup = ~paste0(boro_cd_cod,
                            "<br><b>Since 2017</b><br>",
                            total_injuries, " people injured <br>",
                            total_deaths, " people killed"
                            ))

After making this change:

Re-run the R script
It will regenerate map.html with the hover effect
The boundaries will appear gray and subtle, then highlight bold when you hover
This is a one-parameter addition — very straightforward!