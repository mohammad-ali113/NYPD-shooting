title: "NYC Shooting Data Analysis"
author: "Mohammad Amini"
date: "`r Sys.Date()`"
output: html_document
---

## Introduction
This document imports and explores the NYC shooting incident dataset in a reproducible manner.

## Load Required Libraries
```{r setup, message=FALSE, warning=FALSE}
library(tidyverse)
library(lubridate)
library(hms)
```

## Data Loading and Cleaning 
```{r }
# Define dataset URL
data_url <- "https://data.cityofnewyork.us/api/views/833y-fsy8/rows.csv?accessType=DOWNLOAD"

# Read data into R
df <- read_csv(data_url)

# Display the structure of the data (column names and first few rows)
str(df)

# Check column names to verify the exact names
colnames(df)

# Remove unnecessary columns 
df_clean <- df %>%
  select(-JURISDICTION_CODE, -X_COORD_CD, -Y_COORD_CD, -Latitude, -Longitude, -Lon_Lat)

# Convert OCCUR_DATE to Date format
df_clean <- df_clean %>%
     mutate(OCCUR_DATE = mdy(OCCUR_DATE))

df_clean

```

## Data Visualization 
#1. Number of Incidents by Time Ranges
```{r }

df_time_grouped <- df_clean %>%
  mutate(OCCUR_TIME = hms::as_hms(OCCUR_TIME)) %>%  # Convert to time format
  filter(!is.na(OCCUR_TIME) & OCCUR_TIME != "") %>%  # Remove null & blank values
  mutate(TIME_RANGE = case_when(
    OCCUR_TIME >= hms::as_hms("06:00:00") & OCCUR_TIME < hms::as_hms("12:00:00") ~ "06 AM - 12 PM",
    OCCUR_TIME >= hms::as_hms("12:00:00") & OCCUR_TIME < hms::as_hms("18:00:00") ~ "12 PM - 06 PM",
    OCCUR_TIME >= hms::as_hms("18:00:00") & OCCUR_TIME < hms::as_hms("24:00:00") ~ "06 PM - 12 AM",
    TRUE ~ "12 AM - 06 AM"
  )) %>%
  group_by(TIME_RANGE) %>%
  summarise(Incident_Count = n()) %>%
  arrange(match(TIME_RANGE, c("12 AM - 06 AM", "06 AM - 12 PM", "12 PM - 06 PM", "06 PM - 12 AM")))

df_time_grouped

# Bar Plot
ggplot(df_time_grouped, aes(x = TIME_RANGE, y = Incident_Count, fill = TIME_RANGE)) +
  geom_col() +
  theme_minimal() +
  labs(title = "Number of Incidents by Time of Day", x = "Time Range", y = "Incident Count") +
  scale_fill_brewer(palette = "Set2")
```

## Data Visualization 
#2. Number of Incidents by PERP_AGE_GROUP
```{r }

df_perp_age_grouped <- df_clean %>%
  filter(PERP_AGE_GROUP != "" & !is.na(PERP_AGE_GROUP)) %>%  # Remove null & blank values
  group_by(PERP_AGE_GROUP) %>%
  summarise(Incident_Count = n()) %>%
  arrange(desc(Incident_Count))

df_perp_age_grouped

# Bar Plot
ggplot(df_perp_age_grouped, aes(x = reorder(PERP_AGE_GROUP, -Incident_Count), y = Incident_Count, fill = PERP_AGE_GROUP)) +
  geom_col() +
  theme_minimal() +
  labs(title = "Number of Incidents by Perpetrator Age Group", x = "Age Group", y = "Incident Count") +
  theme(axis.text.x = element_text(angle = 45, hjust = 1)) +
  scale_fill_brewer(palette = "Paired")

# Removing invalid or unknown values
df_perp_age_grouped <- df_clean %>%
  filter(
    PERP_AGE_GROUP != "" & !is.na(PERP_AGE_GROUP) & 
    !PERP_AGE_GROUP %in% c("UNKNOWN", "1020", "1028", "224", "940", "(null)")
  ) %>%
  group_by(PERP_AGE_GROUP) %>%
  summarise(Incident_Count = n()) %>%
  arrange(desc(Incident_Count))

# Bar Plot
ggplot(df_perp_age_grouped, aes(x = reorder(PERP_AGE_GROUP, -Incident_Count), y = Incident_Count, fill = PERP_AGE_GROUP)) +
  geom_col() +
  theme_minimal() +
  labs(title = "Number of Incidents by Perpetrator Age Group", x = "Age Group", y = "Incident Count") +
  theme(axis.text.x = element_text(angle = 45, hjust = 1)) +
  scale_fill_brewer(palette = "Paired")
```


## Data Modeling
```{r }
# Convert PERP_AGE_GROUP to a numeric factor
df_perp_age_grouped$PERP_AGE_GROUP_NUM <- as.numeric(factor(df_perp_age_grouped$PERP_AGE_GROUP))

# Fit a linear regression model
model <- lm(Incident_Count ~ PERP_AGE_GROUP_NUM, data = df_perp_age_grouped)

# Make predictions based on the model
df_perp_age_grouped$Predicted_Incident_Count <- predict(model, newdata = df_perp_age_grouped)

# Calculate the correlation between actual and predicted values
correlation_result <- cor(df_perp_age_grouped$Incident_Count, df_perp_age_grouped$Predicted_Incident_Count)
print(paste("Correlation between Actual and Predicted Incident Count: ", correlation_result))

# Plot the data with prediction line
ggplot(df_perp_age_grouped, aes(x = PERP_AGE_GROUP_NUM, y = Incident_Count)) +
  geom_point() +
  geom_line(aes(x = PERP_AGE_GROUP_NUM, y = Predicted_Incident_Count), color = "red", linetype = "dashed") +
  geom_smooth(method = "lm", se = FALSE, color = "blue") +
  labs(
    title = "Correlation between Incident Count and PERP_AGE_GROUP",
    subtitle = "The red dashed line represents predicted incident counts based on a linear regression model.",
    caption = paste("Correlation coefficient between actual and predicted incident count: ", round(correlation_result, 2)),
    x = "PERP_AGE_GROUP (Numeric)",
    y = "Incident Count"
  ) +
  theme_minimal() +
  theme(axis.text.x = element_text(angle = 45, hjust = 1))  # Rotate x-axis labels for readability
```


## Conclusion
**In conclusion, gun violence peaks during nighttime hours and primarily impacts young adults, highlighting the necessity for targeted safety interventions.**


## Bias
**Reporting Bias**
This bias occurs because not all incidents may have been reported to the authorities.

**Missing Data Bias**
Missing or incomplete data (e.g., missing location or victim information) may occur for various reasons, such as incomplete police reports or data entry errors.We need to do some data cleaning for this bias.

**Data Entry or Human Error Bias**
Errors during the data entry process, such as typos, incorrect geolocation coordinates, or mismatched fields, can introduce biases into the dataset. Like what we saw in the age section. Again we need to find them and remove them from our dataset
