1. One missing value in price (isna().sum() shows 1, and info() shows 96 of 97 non-null)
Decision: Investigate. I need to see which row it is before deciding, and if it can’t be recovered I would exclude it from price analysis.

2. One duplicated row (duplicated().sum() returns 1, and row 96 is True)
Decision: Investigate, then correct by dropping it. I first check it truly matches an earlier row, because an accidental repeat would double-count that record.

3. price is stored as object (info())
Decision: Investigate, then correct. Text in a price column means something non-numeric is in there (symbols, commas, “N/A”), so I check the values before converting to numbers.

4. date is stored as object (info())
Decision: Investigate, then correct. Text dates can’t be sorted or grouped by time, so I check that the formats are consistent and then convert them to real dates.

5. All 6 columns are object (dtypes: object(6))
Decision: Flag. Pandas read everything as text, which suggests messy content, so I won’t trust any column’s type until I’ve checked it.

6. item, city, unit and data_label could have inconsistent text (the code only shows they are object with no nulls)
Decision: Investigate with value_counts(). The code didn’t reveal a problem here, but “messy” data often has spelling or case variants, and I’d only correct them if I actually find some.
