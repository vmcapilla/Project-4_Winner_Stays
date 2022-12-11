# Complete Project 4: Winner Stays: European Football Analysis with Streaks and Countries

# Scenario
You are a junior data analyst working for a business intelligence consultant. You have been at your job for six months, and your boss feels you are ready for more responsibility. He has asked you to lead a project for a brand new client — this will involve everything from defining the business task all the way through presenting your data-driven recommendations. You will choose the topic, ask the right questions, identify a fresh dataset and ensure its integrity, conduct analysis, create compelling data visualizations, and prepare a presentation.

# Main question to answer:
1. What type of company does your client represent, and what are they asking you to accomplish?
2. What are the key factors involved in the business task you are investigating?
3. What type of data will be appropriate for your analysis?
4. Where will you obtain that data?
5. Who is your audience, and what materials will help you present to them effectively?

# 1.0 Ask
After elaborating the first analysis, I would like to improve it, adding new metrics based on functions and adding new visualizations such as maps for practice.

Let's remember the rules: We start in the year 2022, the current champion is Real Madrid after winning the 2001-2002 Champions League Title. From then on and, in every league or cup match, the crown is played. Every time someone manages to defeat him, he conquers the crown and takes it until he is beaten or it reigns a new champion (annually in the final of the Cahmpions League).

# 2.0 Prepare
## 2.1 Loading libraries
```
library(tidyverse)
library(dplyr)
library(tidyr)
library(ggplot2)
library(geomtextpath)
library(lubridate)
library(forcats)
library(viridis)
```

## 2.2 Data loading
```
matches <- read.csv('../input/european-soccer-data/Full_Kaggle_Dataset.csv')
```
## 2.3 Data adjustments
```
matches <- matches %>% drop_na(Home_Score) %>% drop_na(Away_Score)
str(matches)
```
```
matches$Date <- 
    strptime(matches$Date, '%d/%m/%Y')

matches <- dplyr::arrange(matches, Date)

matches <- matches %>% mutate_at(c('Home_Score', 'Away_Score', 'Home_Score_AET', 'Away_Score_AET', 
                                   'Home_Penalties', 'Away_Penalties', 'Home_Points', 'Away_Points' ), as.integer)

str(matches)
```
```
unique(matches$season) #OK
unique(matches$Country) #OK
unique(matches$Competition) #OK
unique(matches$Round)
matches$Round <- str_to_title(matches$Round) #Combinig 'final' and 'FINAL'
matches$Round <- gsub("Semi Finals", "Semi-Finals", matches$Round) #Combining names
unique(matches$Round) #OK
```
## 2.4 Creating New Columns
```
matches$Europe_King <- c('')
matches$New_King <- as.integer(c(''))
str(matches)
```
# 3.0 Process
```
for(i in 1:nrow(matches)) {
  if(i == 1) {
    matches[i, 'Europe_King'] <- 'REAL MADRID'
  }else if(matches[i, 'Round'] == 'Final'&
          matches[i, 'Competition'] == 'uefa-champions-league' & 
          matches[i, 'Home_Points'] > 0) {
    matches[i, 'Europe_King'] <- matches[i, 'Home_Team']
    matches[i, 'New_King'] <- 'YES'
  }else if(matches[i, 'Round'] == 'Final' &
          matches[i, 'Competition'] == 'uefa-champions-league' & 
          matches[i, 'Away_Points'] > 0) {
    matches[i, 'Europe_King'] <- matches[i, 'Away_Team']
    matches[i, 'New_King'] <- 'YES'
  }else if(matches[i, 'Home_Team'] == matches[i-1, 'Europe_King'] &
          matches[i, 'Home_Points'] == 0) {
    matches[i, 'Europe_King'] <- matches[i, 'Away_Team']
    matches[i, 'New_King'] <- 'YES'
  }else if(matches[i, 'Away_Team'] == matches[i-1, 'Europe_King'] &
          matches[i, 'Away_Points'] == 0) {
    matches[i, 'Europe_King'] <- matches[i, 'Home_Team']
    matches[i, 'New_King'] <- 'YES'
  }else if(is.na(matches[i, 'Home_Points']) == TRUE |
          is.na(matches[i, 'Away_Points']) == TRUE) {
    matches[i, 'Europe_King'] <- matches[i-1, 'Europe_King']
  }else if(matches[i, 'Home_Team'] == matches[i-1, 'Europe_King'] &
          matches[i, 'Home_Points'] > 0) {
    matches[i, 'Europe_King'] <- matches[i-1, 'Europe_King']
    matches[i, 'New_King'] <- 'NO'
   }else if(matches[i, 'Away_Team'] == matches[i-1, 'Europe_King'] &
          matches[i, 'Away_Points'] > 0) {
    matches[i, 'Europe_King'] <- matches[i-1, 'Europe_King']
    matches[i, 'New_King'] <- 'NO'
  }else{
    matches[i, 'Europe_King'] <- matches[i-1, 'Europe_King']
  }
}
str(matches)
```
## 3.1 Creating the database by filtering from all matches, from the matches in which the current europe king plays
```
kingmatches <- matches %>% drop_na(New_King)
str(kingmatches)
```
```
kingmatches$Streak <- c(0)

for(i in 1:nrow(kingmatches)) {
  if(i == 1 & kingmatches[i, 'New_King'] == 'NO') {
    kingmatches[i, 'Streak'] <- 2
  }else if(i == 1 & kingmatches[i, 'New_King'] == 'YES') {
    kingmatches[i, 'Streak'] <- 1
  }else if(kingmatches[i, 'New_King'] == 'YES') {
    kingmatches[i, 'Streak'] <- 1  
  }else{
    kingmatches[i, 'Streak'] <- kingmatches[i-1, 'Streak'] +1
  }
}
str(kingmatches)
```
## 3.2 Creating a database that extracts from which country each team is from
```
team_country <-
matches %>% 
    group_by(Home_Team, Country, Europe_King) %>%
    count(Country, Europe_King)  %>%
    filter(Home_Team==Europe_King) %>%
    filter(Country!='europe-uefa')


team_country <- team_country[, -c(3:4)]
team_country <- ungroup(team_country)
colnames(team_country)[1] <- c("Team")
colnames(team_country)[2] <- c("region")
team_country$region <- str_to_title(team_country$region)
team_country$region <- gsub("England", "UK", team_country$region)

str(team_country)
```
# 4.0 Analysis
```
tail((kingmatches$Europe_King), n=1)
tail((matches$Date), n=1)
```
## 4.1 Champions League Winners
```
champions_country <- merge(x=matches, y=team_country, by.x=c('Europe_King'),by.y=c('Team'))

champions_country %>%
    filter(Round =='Final') %>%
    filter(Competition=='uefa-champions-league') %>%
    group_by(Europe_King, region) %>%
    summarise(count=n()) %>%

ggplot(aes(x = fct_reorder(Europe_King, -count), y = count, fill=region)) +
    geom_col(colour="black") +
    labs(title='Fig. 01: Champions Leagues by Team and Country', subtitle='Data from 2002-2022', y='Champions',x='', fill='Country')+
    scale_fill_viridis_d()+
    theme(axis.text.x = element_text(angle = 90, vjust = 0.5, hjust=1))
```
## 4.2 Streaks Evolution
```
kingmatches_country <- merge(x=kingmatches, y=team_country, by.x=c('Europe_King'),by.y=c('Team'))

kingmatches_country %>% 
    mutate(month = format(Date, "%d%m%Y")) %>% 
ggplot(aes(Streak, month)) +
    geom_col(colour="black")+
    coord_flip()+
    labs(title='Fig. 02: Evolution of current Streak as King of Europe', y = '')
```
## 4.3 Top 10 Streaks per Team and grouped by Country
```
kingmatches_country %>%
    group_by(Europe_King, region) %>%
    summarise(Max_Streak=max(Streak,na.rm=T)) %>%
    arrange(desc(Max_Streak)) %>%
    head(10) %>%
ggplot(aes(reorder(Europe_King, -Max_Streak), Max_Streak, fill=region))+
    geom_col(colour="black") +
    scale_fill_viridis(discrete = TRUE) +
    theme(axis.text.x = element_text(angle = 90, vjust = 0.5, hjust=1)) +
    labs(title='Fig. 03: Max Streaks as Kings of Europe', x='', y = 'Max Streak', fill='Country')
```
```
kingmatches_country %>%
    group_by(Europe_King, region) %>%
    summarise(Max_Streak=max(Streak,na.rm=T)) %>%
    arrange(desc(Max_Streak)) %>%
    #head(10) %>%
ggplot(aes(region, Max_Streak, fill=region))+
    geom_violin(colour="black") +
    geom_count()+
    #coord_flip()+
    scale_fill_viridis(discrete = TRUE) +
    #theme(axis.text.x = element_text(angle = 90, vjust = 0.5, hjust=1)) +
    labs(title='Fig. 04: Team distribution by Max Streaks as Kings of Europe and Country', x='', y = 'Max Streak', fill='Country', size='# of teams')
```
## 4.3 Total games as King of Europe by Country and Map representation
```
kingmatches_country %>% 
    group_by(Europe_King, region) %>%
    summarise(count=n()) %>%
    arrange(desc(count)) %>%
    head(10) %>%
ggplot(aes(x = fct_reorder(Europe_King, count), y = count, fill=region)) +
    geom_col(colour="black") +
    coord_curvedpolar()+
    labs(title='Fig. 05: Total games as King of Europe', y='Games',x='', fill='Country')+
    theme(axis.text.x = element_text(angle = 90, vjust = 0.5, hjust=1))+
    scale_fill_viridis(discrete = TRUE)
```
```
countrykings <-
kingmatches_country %>% 
    group_by(region) %>%
    summarise(count=n()) %>%
    arrange(desc(count)) %>%

ggplot(aes(x = fct_reorder(region, count), y = count, fill=region))
```
```
map_data <- map_data("world")
map_data <- left_join(x=map_data, y=countrykings$data, by='region')
map_data <- map_data %>% filter(!is.na(map_data$count))
str(map_data)
```
```
label_data <- map_data %>%
  group_by(region, count) %>%
  summarise(long = mean(long), lat = mean(lat))

map_data %>%
ggplot(aes(x =long, y= lat))+ 
    geom_polygon(aes(group=group, fill=count),colour="black")+
    geom_label(aes(label = count), data=label_data, size = 5, hjust = 0.5)+
    theme(legend.position = "none")+
    labs(title='Fig. 06: Total games as King of Europe', subtitle='By Country of Procedence of the Team', y='',x='')
```
# 5.0 Conclusions

1. Spain has had a great performance, being clearly dominant in relation to other countries (Fig.6)
2. Madrid and Barcelona are the most outstanding teams of the period (Fig. 5), being also the teams that have won the most Champions League titles (Fig. 1)
3. The streaks of Bayern Munich and Inter are surprising, as well as the fact that there are no outstanding teams in France (Fig. 3)

