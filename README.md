# Allstate_Tableau
File repository hosts public data and is intended to be used for the Allstate Tableau competition on Nov. 1, 2019. The following are the steps taken to clean and join the data sets in order to optimize the data for visualization.

```{r}
# Libraries used
library(RCurl)
library(dplyr)
```
## Data Clean-up
### The Issues 
The Austin Animal Shelter exported is unique in its own data set. However, you attempt to blend the data, you run into issues which result in duplication. This is the result of a violation of the (specifically 1NF and 2NF) normalization forms in the data. The columns contain multiple values in a single cell (i.e. location) in which the field includes the address, city, and state. Fortunately, we can address the 1NF violation in Tableau and normalize the data accordingly. In addition, the data sets have no unique identifiers within them, which makes the blending of the data difficult. One method simple method to resolve this issue is to union the tables. Each row would then represent an instance, regardless of whether it is an intake or outcome. However, that makes Tableau do a lot of the heavy lifting when attempting to aggregate the data since there would be an immense number of null values. Instead, we'll do the heavy lifting for Tableau and aggregate the data into 1 table in which every instance is comprised of a corresponding visits (intake and outcome) to the center. 
#### Step 1 - Import Data
While the data was provided by judges for the competition, it was imported into Github for easier accessing. 
```{r}
Intakes <-read.csv(text = getURL("https://raw.githubusercontent.com/javierjr55/Allstate_Tableau/master/Austin%20Animal%20Center%20Shelter%20Intakes.csv"), header = TRUE)
Outcome <- read.csv(text = getURL("https://raw.githubusercontent.com/javierjr55/Allstate_Tableau/master/Austin%20Animal%20Center%20Shelter%20Outcomes.csv"), header = TRUE)
```
#### Step 2 - Create the Rank Value
We know there are going to be issues the moment we join the data. However, we need to identify those issues. The best way to identify the issues is to group the data based on the `animal_id`, order the data within the groups, and then create an index within each group. By applying this to each data set, we are giving ourself an additional value that transcends to both data sets and allows us to join. 
```{r}
# Create ranks for intake instances
Intakes <- Intakes.raw %>% # Call data
  group_by(animal_id) %>% # Call group_by function
  arrange(Intake_Date) %>% # Arrange previous together
  mutate(rank = order(Intake_Date)) # Create rank within group and arrangement

# Repeat for outcome data set
Outcome <- Outcome.raw %>%
  group_by(animal_id) %>%
  arrange(Outcome_Date) %>%
  mutate(rank = order(Outcome_Date))
```
#### Step 3 - The Join
Now that we have a rank/index for each instance within the animals, we can join the data sets! This will allow us to merge together the 1st `Intake_Date` with the 1st `Outcome_Date`. The join will look to make sure the `animal_id` and `rank` values match before combining the tables. We know that not all `animal_id` values will match since there may be some intakes/outcomes outside of the data window. What we want to know which ones they are!
```{r}
# Attempt to join both data sets on newly created rank and animal_id
Animal_Data <- merge(x=Intakes,y=Outcome,by=c("animal_id", "rank"),all=TRUE)
```
#### Step 4 - The Errors
Once the join takes place, we can see there are clearly issues visible in the data. There are 128 rows in which the `Intake_Date` is ahead of the `Outcome_Date`. This is occurring because of that window we discussed earlier. 128 rows may not seem like a lot in a data set that is over 80,000 rows each. However, these are inappropriate joins which may be affecting the rest of the data set. Fortunately, now we know which `animal_id` values won't have an `Intake_Date` because their date was outside data capture. In addition, it is important to note the data has different date lengths. The outcome data set ends on 04/03/2018 4:51 PM, while the intake data set ends on 03/29/2018 6:20 PM. This adds to the issue of animals with the potential to not have an `Intake_Date` but have an `Outcome_Date`. 

We start by isolating the erroneous joins into its own data frame. From there, we extract the `animal_id` values to be able to identify which `animal_id` values are the ones that need modification.
```{r}
# Isolate errors in join
Errors = filter(Animal_Data, as.Date(Animal_Data$Outcome_Date) < as.Date(Animal_Data$Intake_Date))

# Get only Animal_IDs of those who have an outcome date before intake date
Errors_Animal_ID = subset(Errors[!duplicated(Errors$animal_id),], select = c('animal_id'))

# Use join as filter to get rest of outcome data on specific animals
Outcome_first = merge(Errors_Animal_ID, Outcome.raw, by = 'animal_id')

# Trim join just to critical columns
Outcome_first = join[c('animal_id','Outcome_Date')]
```
Once we know which values are the ones that need their rank modified, we mark them by assigning a '1' to an ambiguous column. The fact that we decided to mark them with a '1' is important for an arithmetic reason. By adding it back to the data set with the original rankings, we now know we can easily use the x to subtract that instance from the original ranking. This would then revert the outcomes without an `Intake_Date` to have a rank of 0, which will prevent them from joining to any rank other than 0.
```{r}
# Identify instances in which outcome doesn't have corresponding intake
Outcome_first$x = 1

# Add modified ranks back into the outcomes data set
Outcome_new = merge(Outcome, Outcome_first, by = c('animal_id','Outcome_Date'), all = TRUE)

# Replace nas with 0 for subtraction
Outcome_new$x[is.na(Outcome_new$x)] = 0

# Get modified rank by correcting rank (subtract 1)
Outcome_new$modrank = Outcome_new$rank - Outcome_new$x

# Drop the old rank and x
Outcome_new = select(Outcome_new, -c(rank,x))

# Rename modrank to rank for join
colnames(Outcome_new)[colnames(Outcome_new)=="modrank"] <- "rank"
```
#### Step 5 - The Re-Join
With the modified rank, we are now able to perform the join and ensure each `Intake_Date` lines is joined with its corresponding `Outcome_Date`. This helps us maintain aggregation in the data set while also efficiently utilizing memory in our machine/tools. After validating the join, we find there are no instances in which the `Intake_Date` is greater than the `Outcome_Date`. While there are still many things that could be done to the data, we will reserve most of those for Tableau.
```{r}
# Now we can try to blend on the new ranks since we addressed the issue
new_Animal_Data <- merge(x=Intakes,y=Outcome_new,by=c("animal_id", "rank"),all=TRUE)

Please see Tableau for visualization of data.
```{r}
# Export Data
write.csv(Animal_Data, file = "Animal_Data.csv")

# Uncomment to spot check data and validate
sum(difftime(new_Animal_Data$Outcome_Date,new_Animal_Data$Intake_Date) < 0)
```
See Tableau Public for visualization of data
