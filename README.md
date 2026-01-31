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

## 2. Al Jazeera UN Humanitarian Spending Quiz

Repository URL: https://github.com/AJInteractive/UNHumanitarianSpending.git

### Questions I asked the chatbot
#### Q1: It looks like this site isn't live anymore, and when I view the URL it brings me to the coded in 404.html page. What was it originally coded to do, and what is the benefit of coding in a 404 page?

A: This was an interactive quiz/game created by Al Jazeera Labs titled "Where do we spend more money?" The site tested users' knowledge about annual global humanitarian spending.

The quiz worked by:

Presenting two side-by-side scenarios (e.g., "Humanitarian assistance for Syrian refugees" vs "Providing makeup to celebrities")
Asking users to guess which one costs more money annually
Showing real UN humanitarian funding data compared to consumer spending
Keeping track of the user's score and high score
Displaying funding gaps (money received vs. money still needed) for humanitarian causes
The data came from UN sources and compared serious humanitarian needs (refugee aid, child protection, food security) with consumer spending on things like pet costumes, energy drinks, and cosmetics—highlighting the disparity between humanitarian funding and commercial spending.

Benefits of coding in a 404 page:
Having a custom 404 page provides several important benefits:

User experience - Instead of seeing a confusing browser error, visitors get a styled page that matches the site's branding and clearly explains what happened

Navigation help - A good 404 page can guide users back to working parts of the site or suggest alternatives

Professionalism - It shows attention to detail and makes the site feel more polished

Analytics - You can track when users hit broken links or incorrect URLs to identify and fix problems

SEO benefits - Search engines view proper error handling positively, and it prevents broken pages from damaging your site's reputation

In this case, since the site is no longer hosted on Al Jazeera's servers, the 404 page serves its exact purpose—gracefully handling the "page not found" situation.

#### Q2: How are there different quiz images set up without building multiple html pages?

A: The quiz uses dynamic content loading with JavaScript to show different images without building multiple HTML pages. Here's how it works:

The Single Page Architecture:
1. Static HTML Template (index.html)

Has only TWO section containers: #section1 and #section2
These are empty shells that get filled with content dynamically
The HTML never changes—it's just a template
2. Data Storage (dataset.json)

Contains all quiz data in JSON format
Each entry includes:
question: the quiz question
hint: additional context
value: the monetary amount
image: path to the image file (e.g., "img/L3.jpg")
Funding statistics
3. JavaScript Magic (quizz.js)

The key technique is using jQuery's .css() method to dynamically change background images:

$("#section1").css("background-image", "url(" + leftdata[currentleft].image + ")");
$("#section2").css("background-image", "url(" + rightdata[currentright].image + ")");

How it works:

Loads random quiz items from the JSON data
Extracts the image path from the data (like "img/L3.jpg")
Uses JavaScript to set the CSS background-image property
When user answers, it loads new data and swaps the images again
Preloads next images for smooth transitions
This is called a Single Page Application (SPA) approach—one HTML file dynamically updates its content using JavaScript and data files. Much more efficient than creating dozens of separate HTML pages for each quiz question!

#### Q3: Was the text coded to overlay the images, or did the images already have text overlay?

A: The text is coded to overlay the images through CSS. The images themselves are plain photographs without any text.

Here's how it works:

CSS Overlay Technique:
1. Images as Backgrounds:

#section1 {
  background-image: url(../img/L3.jpg);
  background-size: cover;
  background-position: center center;
}

The images are set as full-screen background images

2. Text Elements Positioned on Top:

.headingone, .headingtwo {
  padding: 10px;
  font-size: 3em;
  font-weight: 900;
  text-shadow: 1px 1px #333333;  /* Shadow for readability */
}

.moneyone, .moneytwo {
  position: absolute;  /* Key: positioned over the image */
  text-align: center;
  top: 65%;           /* 65% down from top */
  left: 0;
  right: 0;
  color: #fb9d04;     /* Orange color */
  font-size: 6em;
}

3. Text Shadow for Readability:
The text-shadow: 1px 1px #333333 ensures white text is readable even on light backgrounds.

This approach is much more flexible than burning text into images because:

Text can be dynamically changed by JavaScript
Text remains searchable and accessible
Easy to update content without re-editing images
Supports multiple languages/translations
Better for SEO and screen readers
The plain photo images are stored in the img folder, and the HTML/CSS layers the quiz questions, hints, and monetary values on top.

#### Q4: What is a .json file? And how was this used to populate the quiz options?

A: JSON stands for "JavaScript Object Notation" - it's a standardized format for storing and exchanging data in a way that's easy for both humans to read and computers to process.

Think of it as a structured filing system where data is organized in a predictable pattern using:

Curly braces {} for objects (like a container)
Square brackets [] for arrays/lists
Key-value pairs like "question": "Healthcare of the Libyans"
Data types: strings (text in quotes), numbers, booleans (true/false)
JSON is language-independent, meaning any programming language can read and write it - making it perfect for data exchange.

How this JSON file populated the quiz:
The dataset.json file is structured with two main arrays:

1. dataone - Humanitarian spending (left side of quiz)
2. datatwo - Consumer/military spending (right side of quiz)

Each entry contains all the data needed for one quiz option:

{
  "question": "Healthcare of the Libyans",
  "hint": "An estimated 1.9 million...",
  "value": "$7m",
  "money": 7,
  "image": "img/L7.jpg",
  "outstandingcash": "$35m",
  "outstanding": 35,
  ...
}

The JavaScript reads this file:

$.getJSON("dataset.json", function(obj) {
  // Random selection from humanitarian data
  leftdata.push(obj.dataone[Math.floor(Math.random() * obj.dataone.length)]);
  
  // Random selection from consumer spending data
  rightdata.push(obj.datatwo[Math.floor(Math.random() * obj.datatwo.length)]);
  
  // Populate the HTML elements
  $(".headingone").html(leftdata[0].question);
  $(".infoone").html(leftdata[0].hint);
  $("#section1").css("background-image", "url(" + leftdata[0].image + ")");
}

The benefits:
Separation of content from code - Update quiz data without touching HTML/JavaScript
Easy to add questions - Just add another object to the array
Scalable - Currently has ~30 options on each side
No database needed - Simple file-based solution for static data
This is why JSON is used everywhere in web development - it's the perfect bridge between data storage and dynamic web content!

## 3. Inside Climate News Human Rights Reports Changes Tracking

Repository URL: https://github.com/InsideClimateNews/2025-08-state-department-human-rights.git

### Questions I asked the chatbot
#### Q1: How does this code use python to search for and analyze key words?

A: This project actually uses R, not Python, to search for and analyze keywords. Here's how it works:

Keyword Search & Analysis Process
1. Web Scraping (scraper.R):

Downloads HTML files from State Department reports (2016-2023)
Extracts the country name from page titles
Pulls report text from HTML elements with class .report__main p
Stores all text in a tibble (R data frame)
2. Keyword Extraction (index.Rmd:32-48):

Uses regex patterns to search for specific terms in each report
Examples: "human rights", "indigenous", "climate change", "corruption", "torture"
Uses str_extract_all() with case-insensitive matching to find all occurrences
Counts total mentions of each term per document
3. Analysis & Aggregation (index.Rmd:52-76):

Groups data by year
Calculates mention rates per 100,000 words (normalized metric)
Creates a summary table showing how keyword mentions have changed over time
The analysis revealed that mentions of "indigenous rights" and "climate change" decreased significantly in the 2024 reports
The key R packages used are tidyverse (for data manipulation) and tidytext/quanteda (for text analysis). The workflow compares trends across 9 years of reports to identify changes in emphasis.

#### Q2: When I view the preview of this file, it brings me to a neatly organized page explaining the code with boxed-in tables and multiple web pages to flip through. Can you find and explain the section of the code that specifically creates boxed data tables and multiple web pages?

A: The Boxed Tables and Multiple Pages Code
Your file uses DataTables, a jQuery plugin that creates interactive, searchable tables with pagination. Here are the key sections:

1. DataTables Library (Lines 2450-2500+)
The HTML includes the complete DataTables JavaScript library, which provides:

Sortable columns - Click column headers to sort
Search functionality - Filter table rows in real-time
Pagination - "Previous/Next" buttons to flip through pages
Styling - The boxed appearance comes from CSS rules like:

table.dataTable {
  width:100%;
  margin:0 auto;
  clear:both;
  border-collapse:separate;
  border-spacing:0;
}

table.dataTable>thead>tr>th,
table.dataTable>tfoot>tr>th {
  padding:10px;
  border-bottom:1px solid rgba(0, 0, 0, 0.3);
}

2. Creating the Data Tables in R
Looking at your index.Rmd:189-197, the actual table creation uses the DT::datatable() function:

datatable()

This R function (from the DT package) wraps the DataTables JavaScript library and automatically:

Generates the boxed table styling
Creates navigation buttons for multiple pages
Adds search boxes
Handles sorting
3. Multiple Pages Implementation
The pagination comes from DataTables' sPaginationType setting (visible in the CSS). When you have more rows than fit on one page, it automatically:

Breaks data into chunks (default 10 rows per page)
Creates "Previous/Next" buttons
Shows "Page X of Y" information
The code you see in index.html:2725 handles the styling of these pagination buttons:

.dataTables_wrapper .dataTables_paginate .paginate_button {
  box-sizing:border-box;
  display:inline-block;
  min-width:1.5em;
  padding:.5em 1em;
  margin-left:2px;
}

In summary: The R code (DT::datatable()) calls the JavaScript DataTables library, which automatically generates the interactive boxed tables with search, sorting, and multi-page navigation all built in.

#### Q3: Could this file be easily repurposed for a different document scraping purpose?

A: Yes, this project is moderately repurposable with varying levels of effort:

Easy to adapt (scraper.R):

The HTML parsing structure relies on two CSS selectors: title for the document name and .report__main p for content
If your new source has predictable HTML structure and similar CSS classes/IDs, you'd mainly need to change:
The file path loop (html/{year})
The CSS selectors (.report__main p → whatever class/ID contains your content)
The country name extraction logic
Example: If scraping newspaper articles with article tags, you'd swap .report__main p for article and adjust the metadata parsing.

More work (index.Rmd):

The keyword analysis is highly specific to human rights language (indigenous, lgbt, torture, climate change, etc.)
For a different topic, you'd need to redefine all ~17 keywords/phrases to match your domain
The data pipeline itself is reusable: extract text → count mentions → aggregate by time/grouping → render interactive tables

What's already reusable:

The tidyverse/rvest web scraping pattern
The str_extract_all() + regex approach for keyword extraction
The aggregation and summarization logic
The DT::datatable() rendering (works with any data frame)

Bottom line: The framework is portable, but the content is specialized. You'd spend ~30% effort adapting the scraper and ~70% on redefining keywords and analysis logic for a new domain.

#### Q4: Where do you code R markdown script, and how does it generate an HTML file?

A: R Markdown scripts (.Rmd files) are typically coded in:

RStudio (most common) — dedicated IDE with built-in Markdown editor and "Knit" button
VS Code with R extensions
Any text editor (the file is just plain text with code blocks)
How it generates HTML:

You write the .Rmd file — contains YAML header, narrative text, and R code blocks (wrapped in ```{r} ... ```)

You click "Knit" (or run rmarkdown::render() in R) — triggers the compilation process:

knitr package executes all R code blocks and captures their output
R's tidyverse, DT, etc. packages generate tables, plots, and data visualizations
The output (tables, stats) gets embedded back into the document
pandoc converts the merged Markdown → HTML:

Markdown text becomes HTML paragraphs, headers, lists
Inline code blocks become <pre> and <code> tags
R-generated content (like DT::datatable()) outputs as HTML <table> tags with embedded JavaScript
Final HTML file contains:

All narrative text as HTML
All R output (tables, charts) as HTML/JavaScript
CSS and JavaScript libraries needed to display interactive elements
In your case, index.Rmd → knit → executes all the R keyword-counting code → DT creates DataTables HTML → index.html is the final output.`)

You click "Knit" (or run rmarkdown::render() in R) — triggers the compilation process:

knitr package executes all R code blocks and captures their output
R's tidyverse, DT, etc. packages generate tables, plots, and data visualizations
The output (tables, stats) gets embedded back into the document
pandoc converts the merged Markdown → HTML

Markdown text becomes .html paragraphs, headers, lists Inline code blocks become pre and code tags
R-generated content (like DT::datatable()) outputs as HTML table tags with embedded JavaScript
Final HTML file contains:

All narrative text as HTML
All R output (tables, charts) as HTML/JavaScript
CSS and JavaScript libraries needed to display interactive elements
In your case, index.Rmd → knit → executes all the R keyword-counting code → DT creates DataTables HTML → index.html is the final output.