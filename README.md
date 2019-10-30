# Allstate_Tableau
File repository hosts public data and is intended to be used for the Allstate Tableau competition on Nov. 1, 2019. The following are the steps taken to join the data in a manner that it is instance complete. While this join is normalized, it is far from the ideal from a data management aspect. This is specifically for visualization. 

```{r}
# Libraries used
library(RCurl)
library(dplyr)
```
## Data Clean-up
### The Issue 
The Austin Animal Shelter data violates of the core principals of normalization theory. The data fails to contain a unique value identifying the individual instance. As a result, we turn to R to create a primary key which could help us join both intake and outcome data sets into 1.
#### Step 1 - Import Data
While the data was provided by judges for the competition, it was imported into Github for easier accessing. This allows me to preserve the integretity of the data itself. 
```{r}
Intakes <-read.csv(text = getURL("https://raw.githubusercontent.com/javierjr55/Allstate_Tableau/master/Austin%20Animal%20Center%20Shelter%20Intakes.csv"), header = TRUE)
Outcome <- read.csv(text = getURL("https://raw.githubusercontent.com/javierjr55/Allstate_Tableau/master/Austin%20Animal%20Center%20Shelter%20Outcomes.csv"), header = TRUE)
```
#### Step 2 - Sort the Data
In this instance, we want to make sure we can sort/order the data by ascending order. The data frame recognizes the date is a mm/dd/yy hh:mm formate and allows us to simply sort the data frame.
```{r}
# Order the data sets by ascending order
Intakes <- Intakes[ order(Intakes$Intake_Date, decreasing = FALSE),]
Outcome <- Outcome[ order(Outcome$Outcome_Date, decreasing = FALSE),]
```
#### Step 3 - Create the Rank Value
In this instance, I essentially grouped the data frame by the animal ID values and 'ranked' them by instance. Essentially, the instances are ordered (again just incase the grouping takes them out of order) by the date and then ranked. In this case, they are ranked on earliest which means the first instance becomes 1 and so on. This helps essentially assign a 'counter' for the number of times an animal is entered into the system again. 
```{r}
Intakes <- Intakes %>% # Call data
  group_by(animal_id) %>% # Call group_by function
  arrange(Intake_Date) %>% # Arrange previous together
  mutate(rank = order(Intake_Date)) # Create rank within group and arrangement
# Repeat for  other data set
Outcome <- Outcome %>%
  group_by(animal_id) %>%
  arrange(Outcome_Date) %>%
  mutate(rank = order(Outcome_Date))
```
#### Step 4 - Creating the Unique Value
With the rank value already in place, we can simply merge the rank value with the animal ID in order to get a unique value through the entire data frame. In other words, with both values being unique to each animal and instance, there should only be one key value in the data frames. This applies to intakes and outcomes. For example, animal 'A3423453' came into the system and left, twice. Therefore, there will be values {A3423453.1, A3423453.2} in both data frames. 
```{r}
# Get 'anima'_id' + '.' + 'rank' value created above to create newly identifiable variable.
# Exmaple: 'A3423453.2'
Intakes = transform(Intakes, key=paste(animal_id, rank, sep="."))
Outcome = transform(Outcome, key=paste(animal_id, rank, sep="."))
```
#### Step 5 - The Join
In order reduce redundancy after the join, I used the newly created primary key for the join. This creates a new dataframe which is at the in/out level. While an intake will initiate an instance, only an outcome will complete the instance. Every instance represents a complete record of a 'pass' by an animal. Considering there are some values that will contain outcomes but no intakes. These animals are the ones that were already in the system prior to the records provided. However, the ones we are unable to track at the ones that were in the system and remained in the system past the alloted time captured in the query.  
```{r}
# Join both data sets on newly created unique value
Animal_Data <- merge(x=Intakes,y=Outcome,by="key",all=TRUE)
# Preview first 5 rows of new join
head(Animal_Data)
```
The new dataset is now ready to be used and explored. While there are certainly plenty more data prep steps that could be handled by R (such as the address, outcomes, etc.), we will save those for Tableau and perform the steps there in forms of calculations considering this is a Tableau competition and not an R competition. 

Please see Tableau for visualization of data.
```{r}
write.csv(Animal_Data, file = "Animal_Data.csv")
```
