Yes, I understand now. You want the **Strategy metric itself** to behave like this Power BI measure:

```DAX
Email Open Last 7 Days =
VAR MaxReadDate =
    CALCULATE(
        MAX(cib_tbl_readership[ReadDateTime]),
        ALL(cib_tbl_readership)
    )
RETURN
CALCULATE(
    COUNTROWS(cib_tbl_readership),
    cib_tbl_readership[ReadDateTime] >= MaxReadDate - 7,
    cib_tbl_readership[ReadDateTime] <= MaxReadDate,
    cib_tbl_readership[Channel] = "Email (Open)"
)
```

So in Strategy, the equivalent idea is:

> Count rows where Channel = Email (Open), and Read Date Time is between Maximum Read Date Time - 7 and Maximum Read Date Time.

Try creating the Strategy metric like this, without `read_7` / `read_30` flag:

```text
Sum(
  Case(
    AND(
      [Channel]@Channel = "Email (Open)",
      [Read Date Time]@ID >= DateAdd([Maximum Read Date Time]@ID, -7),
      [Read Date Time]@ID <= [Maximum Read Date Time]@ID
    ),
    1,
    0
  )
)
```

If `DateAdd()` is not accepted in Strategy metric editor, try direct subtraction:

```text
Sum(
  Case(
    AND(
      [Channel]@Channel = "Email (Open)",
      [Read Date Time]@ID >= [Maximum Read Date Time]@ID - 7,
      [Read Date Time]@ID <= [Maximum Read Date Time]@ID
    ),
    1,
    0
  )
)
```

For **Online Last 7 Days**:

```text
Sum(
  Case(
    AND(
      [Channel]@Channel = "Bloomberg",
      [Read Date Time]@ID >= [Maximum Read Date Time]@ID - 7,
      [Read Date Time]@ID <= [Maximum Read Date Time]@ID
    ),
    1,
    0
  )
)
```

For **Email Open Last 30 Days**:

```text
Sum(
  Case(
    AND(
      [Channel]@Channel = "Email (Open)",
      [Read Date Time]@ID >= [Maximum Read Date Time]@ID - 30,
      [Read Date Time]@ID <= [Maximum Read Date Time]@ID
    ),
    1,
    0
  )
)
```

For **Online Last 30 Days**:

```text
Sum(
  Case(
    AND(
      [Channel]@Channel = "Bloomberg",
      [Read Date Time]@ID >= [Maximum Read Date Time]@ID - 30,
      [Read Date Time]@ID <= [Maximum Read Date Time]@ID
    ),
    1,
    0
  )
)
```

Important: in Strategy, do **not** type the metric name inside the formula box. So do not write:

```text
Email Open =
```

Only paste the expression starting from `Sum(...)`.

Also, instead of typing the attribute names manually, insert them from the object list so Strategy uses the exact form syntax. The main issue in your screenshot is likely the attribute form reference, not the logic.