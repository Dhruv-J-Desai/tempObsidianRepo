Yes — you are right. If the original visual has **Email Open**, **Online**, and also uses **Email Click / Link**, then we should add **Email_Click** as a separate column or include it in the total depending on the original definition.

Since your actual `Channel` values are:

```text
Bloomberg
Email (Click)
Email (Open)
```

Create this missing measure:

```DAX
Email_Click Last 7 Days =
VAR MaxDate =
    CALCULATE(
        MAX(cib_tbl_readership[ReadDateTime]),
        ALL(cib_tbl_readership)
    )
RETURN
CALCULATE(
    COUNTROWS(cib_tbl_readership),
    cib_tbl_readership[ReadDateTime] >= MaxDate - 7,
    cib_tbl_readership[ReadDateTime] <= MaxDate,
    cib_tbl_readership[Channel] = "Email (Click)"
)
```

Then your first visual should probably be:

```text
Title
Email_Click Last 7 Days
Email_Open Last 7 Days
Online Last 7 Days
Grand Total Last 7 Days
```

Rename columns like:

```text
Email_Click → Email_Link
Email_Open  → Email_Open
Online      → Online
Grand Total → Grand_Total
```

Now update Grand Total depending on what the source dashboard expects.

### Option A: Grand Total includes all 3 channels

This is usually the safer definition:

```DAX
Grand Total Last 7 Days =
[Email_Click Last 7 Days]
+ [Email_Open Last 7 Days]
+ [Online Last 7 Days]
```

### Option B: Grand Total only includes Email Open + Online

Use this only if the original screenshot clearly excludes Email Click:

```DAX
Grand Total Last 7 Days =
[Email_Open Last 7 Days]
+ [Online Last 7 Days]
```

For your current data, because `Email (Click)` exists as a channel, I would include it unless the original report intentionally separates link clicks from total readership.

So final setup:

```text
Title
Email_Click Last 7 Days
Email_Open Last 7 Days
Online Last 7 Days
Grand Total Last 7 Days
```

Sort by:

```text
Grand Total Last 7 Days descending
```

Top N filter:

```text
Title → Top N → Top 25 by Grand Total Last 7 Days
```