
# Configuration

You need to follow these steps to configure your RAPIDS deployment before you can extract behavioral features

1. Add your [database credentials](#database-credentials)
2. Choose the [timezone of your study](#timezone-of-your-study)
3. Create your [participants files](#participant-files)
4. Select what [time segments](#time-segments) you want to extract features on
5. Modify your [device data source configuration](#device-data-source-configuration)
6. Select what [sensors and features](#sensor-and-features-to-process) you want to process

When you are done with this configuration, go to [executing RAPIDS](../execution).

!!! hint
    Every time you see `config["KEY"]` or `[KEY]` in these docs we are referring to the corresponding key in the `config.yaml` file.

---
## Database credentials

1. Create an empty file called `#!bash .env` in your RAPIDS root directory
2. Add the following lines and replace your database-specific  credentials (user, password, host, and database):

  ``` yaml
  [MY_GROUP]
  user=MY_USER
  password=MY_PASSWORD
  host=MY_HOST
  port=3306
  database=MY_DATABASE
  ```

!!! warning
    The label `MY_GROUP` is arbitrary but it has to match the following `config.yaml` key:

    ```yaml
    DATABASE_GROUP: &database_group
      MY_GROUP
    ```

!!! note
    You can ignore this step if you are only processing Fitbit data in CSV files.
---

## Timezone of your study

### Single timezone

If your study only happened in a single time zone, select the appropriate code form this [list](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones) and change the following config key. Double check your timezone code pick, for example US Eastern Time is `America/New_York` not `EST`

``` yaml
TIMEZONE: &timezone
  America/New_York
```

### Multiple timezones

Support coming soon.

---

## Participant files

Participant files link together multiple devices (smartphones and wearables) to specific participants and identify them throughout RAPIDS. You can create these files manually or [automatically](#automatic-creation-of-participant-files). Participant files are stored in `data/external/participant_files/pxx.yaml` and follow a unified [structure](#structure-of-participants-files).

!!! note
    The list `PIDS` in `config.yaml` needs to have the participant file names of the people you want to process. For example, if you created `p01.yaml`, `p02.yaml` and `p03.yaml` files in `/data/external/participant_files/ `, then `PIDS` should be:
    ```yaml
    PIDS: [p01, p02, p03] 
    ```

!!! tip
    Attribute *values* of the `[PHONE]` and `[FITBIT]` sections in every participant file are optional which allows you to analyze data from participants that only carried smartphones, only Fitbit devices, or both.

??? hint "Optional: Migrating participants files with the old format"
    If you were using the pre-release version of RAPIDS with participant files in plain text (as opposed to yaml), you can run the following command and your old files will be converted into yaml files stored in `data/external/participant_files/`

    ```bash
    python tools/update_format_participant_files.py
    ```

### Structure of participants files

!!! example "Example of the structure of a participant file"

    In this example, the participant used an android phone, an ios phone, and a fitbit device throughout the study between Apr 23rd 2020 and Oct 28th 2020

    ```yaml
    PHONE:
      DEVICE_IDS: [a748ee1a-1d0b-4ae9-9074-279a2b6ba524, dsadas-2324-fgsf-sdwr-gdfgs4rfsdf43]
      PLATFORMS: [android,ios]
      LABEL: test01
      START_DATE: 2020-04-23
      END_DATE: 2020-10-28
    FITBIT:
      DEVICE_IDS: [fitbit1]
      LABEL: test01
      START_DATE: 2020-04-23
      END_DATE: 2020-10-28
    ```

**For `[PHONE]`**

| Key&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;            | Description                                                                                                                                                                                                                                                                                                                                |
|-------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `[DEVICE_IDS]` | An array of the strings that uniquely identify each smartphone, you can have more than one for when participants changed phones in the middle of the study, in this case, data from all their devices will be joined and relabeled with the last 1 on this list.                                                                           |
| `[PLATFORMS]`  | An array that specifies the OS of each smartphone in  `[DEVICE_IDS]` , use a combination of  `android`  or  `ios`  (we support participants that changed platforms in the middle of your study!). If you have an  `aware_device`  table in your database you can set  `[PLATFORMS]: [multiple]`  and RAPIDS will infer them automatically. |
| `[LABEL]`      | A string that is used in reports and visualizations.                                                                                                                                                                                                                                                                                       |
| `[START_DATE]` | A string with format  `YYY-MM-DD` . Only data collected  *after*  this date will be included in the analysis                                                                                                                                                                                                                               |
| `[END_DATE]`   | A string with format  `YYY-MM-DD` . Only data collected  *before*  this date will be included in the analysis                                                                                                                                                                                                                              |

**For `[FITBIT]`**

| Key&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;              | Description                                                                                               |
|------------------|-----------------------------------------------------------------------------------------------------------|
| `[DEVICE_IDS]`   | An array of the strings that uniquely identify each Fitbit, you can have more than one in case the participant changed devices in the middle of the study, in this case, data from all devices will be joined and relabeled with the last  `device_id`  on this list. |
| `[LABEL]`        | A string that is used in reports and visualizations.                                                                                                                                                                                                                  |
| `[START_DATE]`   | A string with format  `YYY-MM-DD` . Only data collected  *after*  this date will be included in the analysis                                                                                                                                                          |
| `[END_DATE]`     | A string with format  `YYY-MM-DD` . Only data collected  *before*  this date will be included in the analysis                                                                                                                                                         |

### Automatic creation of participant files

You have two options a) use the `aware_device` table in your database or b) use a CSV file. In either case, in your `config.yaml`, set `[PHONE_SECTION][ADD]` or `[FITBIT_SECTION][ADD]` to `TRUE` depending on what devices you used in your study. Set `[DEVICE_ID_COLUMN]` to the name of the column that uniquely identifies each device and include any device ids you want to ignore in `[IGNORED_DEVICE_IDS]`.

=== "aware_device table"

    Set the following keys in your `config.yaml`

    ```yaml
    CREATE_PARTICIPANT_FILES:
      SOURCE:
        TYPE: AWARE_DEVICE_TABLE
        DATABASE_GROUP: *database_group
        CSV_FILE_PATH: ""
        TIMEZONE: *timezone
      PHONE_SECTION:
        ADD: TRUE # or FALSE
        DEVICE_ID_COLUMN: device_id # column name
        IGNORED_DEVICE_IDS: []
      FITBIT_SECTION:
        ADD: TRUE # or FALSE
        DEVICE_ID_COLUMN: fitbit_id # column name
        IGNORED_DEVICE_IDS: []
    ```

    Then run 

    ```bash
    snakemake -j1 create_participants_files
    ```

=== "CSV file"

    Set the following keys in your `config.yaml`. 

    ```yaml
    CREATE_PARTICIPANT_FILES:
      SOURCE:
        TYPE: CSV_FILE
        DATABASE_GROUP: ""
        CSV_FILE_PATH: "your_path/to_your.csv"
        TIMEZONE: *timezone
      PHONE_SECTION:
        ADD: TRUE # or FALSE
        DEVICE_ID_COLUMN: device_id # column name
        IGNORED_DEVICE_IDS: []
      FITBIT_SECTION:
        ADD: TRUE # or FALSE
        DEVICE_ID_COLUMN: fitbit_id # column name
        IGNORED_DEVICE_IDS: []
    ```
    Your CSV file (`[SOURCE][CSV_FILE_PATH]`) should have the following columns but you can omit any values you don't have on each column:

    | Column           | Description                                                                                               |
    |------------------|-----------------------------------------------------------------------------------------------------------|
    | phone device id  | The name of this column has to match `[PHONE_SECTION][DEVICE_ID_COLUMN]`. Separate multiple ids with `;`  |
    | fitbit device id | The name of this column has to match `[FITBIT_SECTION][DEVICE_ID_COLUMN]`. Separate multiple ids with `;`  |
    | pid              | Unique identifiers with the format pXXX (your participant files will be named with this string            |
    | platform         | Use `android`, `ios` or `multiple` as explained above, separate values with `;`            |
    | label            | A human readable string that is used in reports and visualizations.                                       |
    | start_date       | A string with format `YYY-MM-DD`. |
    | end_date         | A string with format `YYY-MM-DD`. |

    !!! example

        ```csv
        device_id,pid,label,platform,start_date,end_date,fitbit_id
        a748ee1a-1d0b-4ae9-9074-279a2b6ba524;dsadas-2324-fgsf-sdwr-gdfgs4rfsdf43,p01,julio,android;ios,2020-01-01,2021-01-01,fitbit1
        4c4cf7a1-0340-44bc-be0f-d5053bf7390c,p02,meng,ios,2021-01-01,2022-01-01,fitbit2
        ```

    Then run 

    ```bash
    snakemake -j1 create_participants_files
    ```

---

## Time Segments

Time segments (or epochs) are the time windows on which you want to extract behavioral features. For example, you might want to process data on every day, every morning, or only during weekends. RAPIDS offers three categories of time segments that are flexible enough to cover most use cases: **frequency** (short time windows every day), **periodic** (arbitrary time windows on any day), and **event** (arbitrary time windows around events of interest). See also our [examples](#segment-examples).

=== "Frequency Segments"

    These segments are computed on every day and all have the same duration (for example 30 minutes). Set the following keys in your `config.yaml`

    ```yaml
    TIME_SEGMENTS: &time_segments
      TYPE: FREQUENCY
      FILE: "data/external/your_frequency_segments.csv"
      INCLUDE_PAST_PERIODIC_SEGMENTS: FALSE
    ```

    The file pointed by `[TIME_SEGMENTS][FILE]` should have the following format and can only have 1 row.

    | Column | Description                                                          |
    |--------|----------------------------------------------------------------------|
    | label  | A string that is used as a prefix in the name of your time segments   |
    | length | An integer representing the duration of your time segments in minutes |

    !!! example

        ```csv
        label,length
        thirtyminutes,30
        ```
        
        This configuration will compute 48 time segments for every day when any data from any participant was sensed. For example:

        ```csv
        start_time,length,label
        00:00,30,thirtyminutes0000
        00:30,30,thirtyminutes0001
        01:00,30,thirtyminutes0002
        01:30,30,thirtyminutes0003
        ...
        ```

=== "Periodic Segments"

    These segments can be computed every day, or on specific days of the week, month, quarter, and year. Their minimum duration is 1 minute but they can be as long as you want. Set the following keys in your `config.yaml`.

    ```yaml
    TIME_SEGMENTS: &time_segments
      TYPE: PERIODIC
      FILE: "data/external/your_periodic_segments.csv"
      INCLUDE_PAST_PERIODIC_SEGMENTS: FALSE # or TRUE
    ```

    If `[INCLUDE_PAST_PERIODIC_SEGMENTS]` is set to `TRUE`, RAPIDS will consider instances of your segments back enough in the past as to include the first row of data of each participant. For example, if the first row of data from a participant happened on Saturday March 7th 2020 and the requested segment duration is 7 days starting on every Sunday, the first segment to be considered would start on Sunday March 1st if `[INCLUDE_PAST_PERIODIC_SEGMENTS]` is `TRUE` or on Sunday March 8th if `FALSE`.

    The file pointed by `[TIME_SEGMENTS][FILE]` should have the following format and can have multiple rows.

    | Column        | Description                                                                                                                                                                                                   |
    |---------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
    | label         | A string that is used as a prefix in the name of your time segments. It has to be **unique** between rows                                                                                                      |
    | start_time    | A string with format `HH:MM:SS` representing the starting time of this segment on any day                                                                                                                                 |
    | length        | A string representing the length of this segment.It can have one or more of the following strings **`XXD XXH XXM XXS`** to represent days, hours, minutes and seconds. For example `7D 23H 59M 59S`                        |
    | repeats_on    | One of the follow options `every_day`, `wday`, `qday`, `mday`, and `yday`. The last four represent a week, quarter, month and year day                                                                        |
    | repeats_value | An integer complementing `repeats_on`. If you set `repeats_on` to `every_day` set this to `0`, otherwise `1-7` represent a `wday` starting from Mondays, `1-31` represent a `mday`, `1-91` represent a `qday`, and `1-366` represent a `yday` |

    !!! example

        ```csv
        label,start_time,length,repeats_on,repeats_value
        daily,00:00:00,23H 59M 59S,every_day,0
        morning,06:00:00,5H 59M 59S,every_day,0
        afternoon,12:00:00,5H 59M 59S,every_day,0
        evening,18:00:00,5H 59M 59S,every_day,0
        night,00:00:00,5H 59M 59S,every_day,0
        ```

        This configuration will create five segments instances (`daily`, `morning`, `afternoon`, `evening`, `night`) on any given day (`every_day` set to 0). The `daily` segment will start at midnight and will last `23:59:59`, the other four segments will start at 6am, 12pm, 6pm, and 12am respectively and last for `05:59:59`. 

=== "Event segments"

    These segments can be computed before or after an event of interest (defined as any UNIX timestamp). Their minimum duration is 1 minute but they can be as long as you want. The start of each segment can be shifted backwards or forwards from the specified timestamp. Set the following keys in your `config.yaml`.

    ```yaml
    TIME_SEGMENTS: &time_segments
      TYPE: EVENT
      FILE: "data/external/your_event_segments.csv"
      INCLUDE_PAST_PERIODIC_SEGMENTS: FALSE # or TRUE
    ```

    The file pointed by `[TIME_SEGMENTS][FILE]` should have the following format and can have multiple rows.

    | Column        | Description                                                                                                                                                                                                   |
    |---------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
    | label         | A string that is used as a prefix in the name of your time segments. If labels are unique, every segment is independent; if two or more segments have the same label, their data will be grouped when computing auxiliary data for features like the `most frequent contact` for calls (the most frequent contact will be computed across all these segments). There cannot be two *overlaping* event segments with the same label (RAPIDS will throw an error)                                                                                                      |
    | event_timestamp    | A UNIX timestamp that represents the moment an event of interest happened (clinical relapse, survey, readmission, etc.). The corresponding time segment will be computed around this moment using `length`, `shift`, and `shift_direction`                                                                                            |
    | length        | A string representing the length of this segment. It can have one or more of the following keys `XXD XXH XXM XXS` to represent a number of days, hours, minutes, and seconds. For example `7D 23H 59M 59S`                        |
    | shift    | A string representing the time shift from `event_timestamp`. It can have one or more of the following keys `XXD XXH XXM XXS` to represent a number of days, hours, minutes and seconds. For example `7D 23H 59M 59S`. Use this value to  change the start of a segment with respect to its `event_timestamp`. For example, set this variable to `1H` to create a segment that starts 1 hour from an event of interest (`shift_direction` determines if it's before or after).                                                                        |
    | shift_direction | An integer representing whether the `shift` is before (`-1`) or after (`1`) an `event_timestamp` |
    |device_id| The device id (smartphone or fitbit) to whom this segment belongs to. You have to create a line in this event segment file for each event of a participant that you want to analyse. If you have participants with multiple device ids you can choose any of them|

    !!! example
        ```csv
        label,event_timestamp,length,shift,shift_direction,device_id
        stress1,1587661220000,1H,5M,1,a748ee1a-1d0b-4ae9-9074-279a2b6ba524
        stress2,1587747620000,4H,4H,-1,a748ee1a-1d0b-4ae9-9074-279a2b6ba524
        stress3,1587906020000,3H,5M,1,a748ee1a-1d0b-4ae9-9074-279a2b6ba524
        stress4,1584291600000,7H,4H,-1,a748ee1a-1d0b-4ae9-9074-279a2b6ba524
        stress5,1588172420000,9H,5M,-1,a748ee1a-1d0b-4ae9-9074-279a2b6ba524
        mood,1587661220000,1H,0,0,a748ee1a-1d0b-4ae9-9074-279a2b6ba524
        mood,1587747620000,1D,0,0,a748ee1a-1d0b-4ae9-9074-279a2b6ba524
        mood,1587906020000,7D,0,0,a748ee1a-1d0b-4ae9-9074-279a2b6ba524
        ```
        
        This example will create eight segments for a single participant (`a748ee1a...`), five independent `stressX` segments with various lengths (1,4,3,7, and 9 hours). Segments `stress1`, `stress3`, and `stress5` are shifted forwards by 5 minutes and `stress2` and `stress4` are shifted backwards by 4 hours (that is, if the `stress4` event happened on March 15th at 1pm EST (`1584291600000`), the time segment will start on that day at 9am and end at 4pm). 
        
        The three `mood` segments are 1 hour, 1 day and 7 days long and have no shift. In addition, these `mood` segments are grouped together, meaning that although RAPIDS will compute features on each one of them, some necessary information to compute a few of such features will be extracted from all three segments, for example the phone contact that called a participant the most or the location clusters visited by a participant.

### Segment Examples

=== "5-minutes"
    Use the following `Frequency` segment file to create 288 (12 * 60 * 24) 5-minute segments starting from midnight of every day in your study
    ```csv
    label,length
    fiveminutes,5
    ```
=== "Daily"
    Use the following `Periodic` segment file to create daily segments starting from midnight of every day in your study
    ```csv
    label,start_time,length,repeats_on,repeats_value
    daily,00:00:00,23H 59M 59S,every_day,0
    ```
=== "Morning"
    Use the following `Periodic` segment file to create morning segments starting at 06:00:00 and ending at 11:59:59 of every day in your study
    ```csv
    label,start_time,length,repeats_on,repeats_value
    morning,06:00:00,5H 59M 59S,every_day,0
    ```
=== "Overnight"
    Use the following `Periodic` segment file to create overnight segments starting at 20:00:00 and ending at 07:59:59 (next day) of every day in your study
    ```csv
    label,start_time,length,repeats_on,repeats_value
    morning,20:00:00,11H 59M 59S,every_day,0
    ```
=== "Weekly"
    Use the following `Periodic` segment file to create **non-overlapping** weekly segments starting at midnight of every **Monday** in your study
    ```csv
    label,start_time,length,repeats_on,repeats_value
    weekly,00:00:00,6D 23H 59M 59S,wday,1
    ```
    Use the following `Periodic` segment file to create **overlapping** weekly segments starting at midnight of **every day** in your study
    ```csv
    label,start_time,length,repeats_on,repeats_value
    weekly,00:00:00,6D 23H 59M 59S,every_day,0
    ```
=== "Week-ends"
    Use the following `Periodic` segment file to create week-end segments starting at midnight of every **Saturday** in your study
    ```csv
    label,start_time,length,repeats_on,repeats_value
    weekend,00:00:00,1D 23H 59M 59S,wday,6
    ```
=== "Around surveys"
    Use the following `Event` segment file to create two 2-hour segments that start 1 hour before surveys answered by 3 participants
    ```csv
    label,event_timestamp,length,shift,shift_direction,device_id
    survey1,1587661220000,2H,1H,-1,a748ee1a-1d0b-4ae9-9074-279a2b6ba524
    survey2,1587747620000,2H,1H,-1,a748ee1a-1d0b-4ae9-9074-279a2b6ba524
    survey1,1587906020000,2H,1H,-1,rqtertsd-43ff-34fr-3eeg-efe4fergregr
    survey2,1584291600000,2H,1H,-1,rqtertsd-43ff-34fr-3eeg-efe4fergregr
    survey1,1588172420000,2H,1H,-1,klj34oi2-8frk-2343-21kk-324ljklewlr3
    survey2,1584291600000,2H,1H,-1,klj34oi2-8frk-2343-21kk-324ljklewlr3
    ```
--- 
## Device Data Source Configuration

You might need to modify the following config keys in your `config.yaml` depending on what devices your participants used and where you are storing your data. You can ignore `[PHONE_DATA_CONFIGURATION]` or `[FITBIT_DATA_CONFIGURATION]` if you are not working with either devices.

=== "Phone"

    The relevant `config.yaml` section looks like this by default:

    ```yaml
    PHONE_DATA_CONFIGURATION:
      SOURCE: 
        TYPE: DATABASE
        DATABASE_GROUP: *database_group
        DEVICE_ID_COLUMN: device_id # column name
      TIMEZONE: 
        TYPE: SINGLE # SINGLE (MULTIPLE support coming soon)
        VALUE: *timezone

    ```

    **Parameters for `[PHONE_DATA_CONFIGURATION]`**

    | Key                  | Description                                                                                                                |
    |---------------------|----------------------------------------------------------------------------------------------------------------------------|
    | `[SOURCE] [TYPE]`             | Only `DATABASE` is supported (phone data will be pulled from a database)                                                   |
    | `[SOURCE] [DATABASE_GROUP]`   | `*database_group`  points to the value defined before in  [Database credentials](#database-credentials)    |
    | `[SOURCE] [DEVICE_ID_COLUMN]` | A column that contains strings that uniquely identify smartphones. For data collected with AWARE this is usually  `device_id` |
    | `[TIMEZONE] [TYPE]`             | Only `SINGLE` is supported for now                                                                                                |
    | `[TIMEZONE] [VALUE]`            | `*timezone`  points to the value defined before in  [Timezone of your study](#timezone-of-your-study)          |

=== "Fitbit"

    The relevant `config.yaml` section looks like this by default:

      ```yaml
      FITBIT_DATA_CONFIGURATION:
        SOURCE: 
          TYPE: DATABASE # DATABASE or FILES (set each [FITBIT_SENSOR][TABLE] attribute with a table name or a file path accordingly)
          COLUMN_FORMAT: JSON # JSON or PLAIN_TEXT
          DATABASE_GROUP: *database_group
          DEVICE_ID_COLUMN: device_id # column name
        TIMEZONE: 
          TYPE: SINGLE # Fitbit devices don't support time zones so we read this data in the timezone indicated by VALUE 
          VALUE: *timezone

      ```

      **Parameters for For `[FITBIT_DATA_CONFIGURATION]`**

      | Key                         | Description                                                                                                                                                       |
      |------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------|
      | `[SOURCE]` `[TYPE]`             | `DATABASE` or `FILES`  (set each `[FITBIT_SENSOR]` `[TABLE]` attribute accordingly with a table name or a file path)                                                                                         |
      | `[SOURCE]` `[COLUMN_FORMAT]`    | `JSON` or `PLAIN_TEXT`. Column format of the source data. If you pulled your data directly from the Fitbit API the column containing the sensor data will be in `JSON` format                                   |
      | `[SOURCE]` `[DATABASE_GROUP]`   | `*database_group`  points to the value defined before in  [Database credentials](#database-credentials). Only used if  `[TYPE]`  is  `DATABASE` . |
      | `[SOURCE]` `[DEVICE_ID_COLUMN]` | A column that contains strings that uniquely identify Fitbit devices.                                                                                                |
      | `[TIMEZONE]` `[TYPE]`             | Only `SINGLE` is supported  (Fitbit devices always store data in local time).                                                                                 |
      | `[TIMEZONE]` `[VALUE]`            | `*timezone`  points to the value defined before in  [Timezone of your study](#timezone-of-your-study)                                             |

---

## Sensor and Features to Process

Finally, you need to modify the `config.yaml` section of the sensors you want to extract behavioral features from. All sensors follow the same naming nomenclature (`DEVICE_SENSOR`) and parameter structure which we explain in the [Behavioral Features Introduction](../../features/feature-introduction/). 

!!! done
    Head over to [Execution](../execution/) to learn how to execute RAPIDS.