# Reproducible Research: Peer Assessment 1


## Loading and preprocessing the data
The following commands will unpack the csv from the zip file, read it into the 
`rawData` data frame and then generate a `processedData` data frame, which
contains the following fields:

- **steps**: Number of steps taking in a 5-minute interval (missing values are coded as `NA`)

- **intervStart**: Date and time (in `POSIXlt` format) of the day and start of 5-minute interval where the measurement was taken

- **intervEnd**: Date and time (in `POSIXlt` format) of the day and end of 5-minute interval where the measurement was taken. By definition is equal to `intervStart` plus 4 minutes and 59 seconds.

- **dateMeasured**: `Date` when the measurement was taken, makes some of the later processings easier.

- **rawInterv**: Formatted text string of the hour and minutes of the interval start, for labeling.



```r
rawData = read.csv(unz("activity.zip", "activity.csv"))
processedData = data.frame(steps = rawData$steps,
                           intervStart = as.POSIXlt(sprintf("%s %d:%d",
                                    rawData$date, rawData$interval %/% 100, 
                                    rawData$interval %% 100),
                                    format="%Y-%m-%d %H:%M"))
processedData$intervStart = as.POSIXlt(processedData$intervStart)
processedData$intervEnd = as.POSIXlt(processedData$intervStart + (60*4) + 59)
processedData$dateMeasured = as.Date(rawData$date)
processedData$rawInterv = sprintf("%02.0f:%02.0f", rawData$interval %/% 100,
                                  rawData$interval %% 100)
```


## What is mean total number of steps taken per day?
The following code calculates the total number of steps each day, discarding `NA`s and plots that in a histogram:

```r
require(plyr);
require(ggplot2);
require(scales);
stepsPerDay = ddply(processedData, .(dateMeasured), summarize, totalSteps=sum(steps, na.rm=TRUE))
g = ggplot(stepsPerDay, aes(dateMeasured, totalSteps)) + 
    geom_bar(stat="identity", fill="dark blue") + xlab("Date") + ylab("Steps") +
    ggtitle("Steps per day") +
    theme(axis.text.x = element_text(angle = 90)) +
    geom_hline(aes(yintercept=mean(stepsPerDay$totalSteps), linetype="Mean"),
               color="dark green", show_guide=TRUE)+
    geom_hline(aes(yintercept=median(stepsPerDay$totalSteps), linetype="Median"),
               color="dark green", show_guide=TRUE)+
    scale_x_date(labels=date_format("%m/%d"), breaks=date_breaks("1 day"))+
    scale_linetype_manual(name="Legend", labels=c("Mean", "Median"),
                          values=c("Mean" = 1, "Median" = 2))
plot(g)
```

![plot of chunk stepsDayHist](figure/stepsDayHist.png) 

The mean total number of steps taken per day is **9354.2295**.
The median total number of steps taken per day is **10395**.

## What is the average daily activity pattern?
The following code calculates and plots the average number of steps per 5-minute
interval across all days of the dataset. Missing data is discarded.

```r
stepsPerInterv = ddply(processedData, .(rawInterv), summarize, avgSteps = mean(steps, na.rm=TRUE))
xLabels = stepsPerInterv$rawInterv[seq.int(1, 288, 6)]
g = ggplot(stepsPerInterv, aes(x=seq.int(1, 288), y=avgSteps)) +
    geom_line(color="black") + xlab("Interval") + ylab("Avg Steps") +
    ggtitle("Average Steps each 5 minutes") +
    theme(axis.text.x = element_text(angle = 90)) +
    scale_x_discrete(breaks = seq.int(1, 288, 6), labels = xLabels) +
    scale_y_continuous(breaks=c(0, 50, 100, 150, 200, max(stepsPerInterv$avgSteps)),
                labels = c("0", "50", "100", "150", "200",
                        sprintf("%0.4f", max(stepsPerInterv$avgSteps))))+
    geom_hline(yintercept=max(stepsPerInterv$avgSteps), color="red")
plot(g)
```

![plot of chunk stepsPerIntervLine](figure/stepsPerIntervLine.png) 

The 5-minute interval starting at **08:35** has
the maximum average number of steps, at **206.1698**.

## Imputing missing values
There is a total of **2304** missing values on the dataset.

Further analysing the data, it can be seen that it's not some measurements that are 
missing, it's entire days (as shown in the following graph).

```r
g = ggplot(processedData, aes(x=dateMeasured, y=rawInterv)) + 
    xlab("Date") + ylab("Interval") + ggtitle("Missing step measurements") +
    theme(axis.text.x = element_text(angle = 90)) +
    scale_x_date(labels=date_format("%m/%d"), breaks=date_breaks("1 day")) +
    scale_y_discrete(breaks =  xLabels) +
    geom_point(aes(color=ifelse(is.na(steps), "NA", "Value"))) +
    scale_color_manual(name="NA ?", values=c("red", "grey"))
plot(g)
```

![plot of chunk missingValues](figure/missingValues.png) 

So, we'll impute based on the weekday, applying the average distribution of
steps per interval for that day.

- Calculate Steps per Week Day, *discarding days with no measurements* (so as not to pull the average down).

```r
stepsPerWeekDay = ddply(stepsPerDay[stepsPerDay$totalSteps > 0,], 
                        .(weekDay=as.factor(weekdays(dateMeasured))), summarize, 
                        avgSteps = mean(totalSteps))
stepsPerWeekDay$weekDay= factor(stepsPerWeekDay$weekDay, 
                    levels(stepsPerWeekDay$weekDay)[c(1, 5, 7, 2, 3, 6, 4)])
# Let's take a look at it
ggplot(stepsPerWeekDay, aes(x=weekDay, y=avgSteps)) +
    geom_bar(stat="identity") +
    ylab("Avg # of Steps") + xlab("Day of week (portuguese)") +
    ggtitle("Average Steps per day of the week")
```

![plot of chunk stepsPerWeekDay](figure/stepsPerWeekDay.png) 

- Calculate the weight of each 5-minute interval for each week day, and observe 
that patterns are different each day.

```r
weightPerInterv = with(processedData, data.frame(steps=steps, 
                                weekDay=as.factor(weekdays(dateMeasured)),
                                rawInterv=rawInterv))
weightPerInterv = ddply(weightPerInterv, .(rawInterv, weekDay),
                        summarize, avgSteps = mean(steps, na.rm=TRUE))
weightPerInterv$weekDay= factor(weightPerInterv$weekDay, 
                    levels(weightPerInterv$weekDay)[c(1, 5, 7, 2, 3, 6, 4)])
lapply(levels(stepsPerWeekDay$weekDay), function(x) {
    weightPerInterv[levels(weightPerInterv$weekDay) == x, 3] <<-
        weightPerInterv[levels(weightPerInterv$weekDay) == x, 3] /
        stepsPerWeekDay[levels(stepsPerWeekDay$weekDay) == x,2]
})
```

```r
g=ggplot(weightPerInterv, aes(as.numeric(rawInterv), avgSteps)) +
    geom_path(aes(color=weekDay, group=weekDay)) + 
    facet_grid(weekDay~., scales="free_y", space="free_y") +
    xlab("5-Minute interval") + ylab("Proportion of steps") +
    ggtitle("Distribution of steps across the day per day of the week") +
    theme(axis.text.x = element_text(angle = 90), legend.position="top") +
    scale_x_discrete(breaks=seq.int(1, 288, 6), labels=xLabels)
plot(g)
```

![plot of chunk weightDistribution](figure/weightDistribution.png) 

- For each day with no recorded steps, fill it with the average # of steps for that
weekday distributed according to the average # of steps at each 5-minute interval

```r
options(scipen=1)
imputedData = processedData
nullDays = unique(processedData[is.na(processedData$steps), 4])
lapply(nullDays, function(x) {
    imputedData[imputedData$dateMeasured == x, 1] <<- as.integer(round(
        stepsPerWeekDay[stepsPerWeekDay$weekDay == weekdays(x),2] * 
        weightPerInterv[weightPerInterv$weekDay == weekdays(x),3]))
})
```

```r
imputedStepsPerDay = ddply(imputedData, .(dateMeasured), summarize, 
                           totalSteps=sum(steps, na.rm=TRUE))
g = ggplot(imputedStepsPerDay, aes(dateMeasured, totalSteps)) + 
    geom_bar(stat="identity", fill="dark blue") + xlab("Date") + ylab("Steps") +
    ggtitle("Steps per day") +
    theme(axis.text.x = element_text(angle = 90)) +
    geom_hline(aes(yintercept=mean(imputedStepsPerDay$totalSteps), linetype="Mean"),
               color="dark green", show_guide=TRUE)+
    geom_hline(aes(yintercept=median(imputedStepsPerDay$totalSteps), linetype="Median"),
               color="dark green", show_guide=TRUE)+
    scale_x_date(labels=date_format("%m/%d"), breaks=date_breaks("1 day"))+
    scale_linetype_manual(name="Legend", labels=c("Mean", "Median"),
                          values=c("Mean" = 1, "Median" = 2))
plot(g)
```

![plot of chunk imputedStepsHist](figure/imputedStepsHist.png) 

After imputation, the mean total number of steps taken per day is **10821.082**.
After imputation, the median total number of steps taken per day is **11015**.

Both these numbers are larger than before, therefore we can conclude that the chosen
imputation strategy **increased** the total number of steps taken.

## Are there differences in activity patterns between weekdays and weekends?

Create a new factor variable in the dataset with two levels – “weekday” and 
“weekend” indicating whether a given date is a weekday or weekend day, and plot 
a comparison between 5-minute intervals on weekdays and weekends.


```r
imputedData$weekDay= as.factor(ifelse(
    imputedData$intervStart$wday %in% c(0,6), "weekend", "work day"))
stepsPerInterv2 = ddply(imputedData, .(rawInterv, weekDay),
                       summarize, avgSteps = mean(steps))
stepsPerInterv2$rawInterv = as.factor(stepsPerInterv2$rawInterv)
g = ggplot(stepsPerInterv2, aes(as.numeric(rawInterv), avgSteps, color=weekDay)) + 
    geom_line(aes(group=weekDay)) + facet_wrap(~weekDay, 2, 1, as.table=FALSE) +
    guides(color=FALSE) +
    xlab("5-Minute interval") + ylab("Average steps") +
    ggtitle("Distribution of steps across the day on workdays and weekends") +
    theme(axis.text.x = element_text(angle = 90)) +
    scale_x_discrete(breaks=seq.int(1, 288, 6), labels=xLabels)
plot(g)
```

![plot of chunk weekDaysxWeekEnds](figure/weekDaysxWeekEnds.png) 
