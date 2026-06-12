I want the Power BI model relationships to match the white ERD exactly.

Please do not infer extra relationships from the DDL only. Create the relationships based on the ERD lines and keys below.

Use these relationships:

1. Lookup_CountryToRegion.CountryCode
   → Tbl_Readership.CountryCode
   Cardinality: One-to-many
   One side: Lookup_CountryToRegion
   Many side: Tbl_Readership

2. Tbl_Dim_Doc.DocID
   → Tbl_Readership.DocID
   Cardinality: One-to-many
   One side: Tbl_Dim_Doc
   Many side: Tbl_Readership

3. Tbl_Dim_Doc.DocID
   → Tbl_ProductTag_Rpt.DocID
   Cardinality: One-to-many
   One side: Tbl_Dim_Doc
   Many side: Tbl_ProductTag_Rpt

4. Lookup_Strategist.StrategistID
   → Tbl_Dim_Doc.StrategistID
   Cardinality: One-to-many
   One side: Lookup_Strategist
   Many side: Tbl_Dim_Doc

5. Lookup_Strategist.StrategistID
   → Tbl_MeetingTracker.StrategistID
   Cardinality: One-to-many
   One side: Lookup_Strategist
   Many side: Tbl_MeetingTracker

6. Tbl_Dim_Reader.ReaderID
   → Tbl_Readership.ReaderID
   Cardinality: One-to-many
   One side: Tbl_Dim_Reader
   Many side: Tbl_Readership

7. Tbl_Dim_Reader.ReaderID
   → Tbl_BMContacts.ReaderID
   Cardinality: One-to-many
   One side: Tbl_Dim_Reader
   Many side: Tbl_BMContacts

8. Lookup_RegionByClientCountry.ClientCountryStateProv
   → Tbl_MeetingTracker.ClientCountryStateProv
   Cardinality: One-to-many
   One side: Lookup_RegionByClientCountry
   Many side: Tbl_MeetingTracker

9. Lookup_AccountProperName_ClientType.AccountID
   → Tbl_MeetingTracker.AccountID
   Cardinality: One-to-many
   One side: Lookup_AccountProperName_ClientType
   Many side: Tbl_MeetingTracker

10. Lookup_AccountProperName_ClientType.AccountProperName
    → Tbl_BMContacts.Account
    Cardinality: One-to-many
    One side: Lookup_AccountProperName_ClientType
    Many side: Tbl_BMContacts

Important:
- Do not create a direct relationship between Tbl_Readership and Tbl_BMContacts.
- Do not create a direct relationship between Tbl_Readership and Lookup_Strategist.
- Readership reaches Strategist through Tbl_Dim_Doc.
- Readership and BMContacts are related indirectly through Tbl_Dim_Reader.
- Use single-direction filtering from the lookup/dimension table to the fact/detail table unless the ERD specifically requires bidirectional filtering.
- Make sure the key columns on the one side have unique values.