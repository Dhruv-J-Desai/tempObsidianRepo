The model is still missing the BMContacts to Account lookup relationship from the white ERD.

Please add this relationship:

cib_lookup_accountpropername_clienttype[AccountProperName]
→ cib_tbl_bmcontacts[Account]

Cardinality: One-to-many
One side: cib_lookup_accountpropername_clienttype
Many side: cib_tbl_bmcontacts
Cross-filter direction: Single

Do not create a direct relationship between Readership and BMContacts.