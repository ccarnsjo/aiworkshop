# ABL Business Entity Architecture Pattern

## Overview

The Business Entity pattern separates user-interface code from database operations. The UI exchanges ProDataSets with business entities; entities encapsulate data access and validation; the database remains the persistent store.

## Architecture

- **UI layer:** Collects user input and displays results. It calls entity methods and should not perform database access itself.
- **Business entity layer:** Inherits from `OpenEdge.BusinessLogic.BusinessEntity`, provides read/create/update/delete operations, and applies validation rules.
- **Database layer:** Exposes data through entity data sources.

An `EntityFactory` can lazily instantiate and reuse entity objects. A dataset include defines the temp-table and ProDataSet shape shared by the entity and its clients.

## Dataset definition

A dataset include declares a temp-table, its fields, a primary index, and the dataset containing it. `BEFORE-TABLE` supports change tracking for updates. Keep the temp-table schema compatible with the database fields used by the entity.

```abl
DEFINE TEMP-TABLE ttCustomer BEFORE-TABLE bttCustomer
    FIELD CustNum AS INTEGER
    FIELD Name AS CHARACTER
    INDEX CustNum IS PRIMARY UNIQUE CustNum.

DEFINE DATASET dsCustomer FOR ttCustomer.
```

## Business entity setup

A business entity passes its dataset handle to the superclass constructor and configures data sources and skip-list entries.

```abl
CLASS business.CustomerEntity INHERITS BusinessEntity USE-WIDGET-POOL:
    {business/CustomerDataset.i}
    DEFINE DATA-SOURCE srcCustomer FOR Customer.

    CONSTRUCTOR PUBLIC CustomerEntity():
        SUPER(DATASET dsCustomer:HANDLE).
        VAR HANDLE[1] hDataSourceArray = DATA-SOURCE srcCustomer:HANDLE.
        VAR CHARACTER[1] cSkipListArray = [""].
        THIS-OBJECT:ProDataSource = hDataSourceArray.
        THIS-OBJECT:SkipList = cSkipListArray.
    END CONSTRUCTOR.
END CLASS.
```

## CRUD patterns

Read methods use `OUTPUT DATASET` and return a success indicator after the superclass fills the dataset. Create, update, and delete methods accept `INPUT-OUTPUT DATASET` and pass it by reference to the matching superclass method. Enable temp-table change tracking before modifying rows that will be persisted.

```abl
METHOD PUBLIC LOGICAL GetCustomerByNumber(INPUT ipiCustNum AS INTEGER,
                                          OUTPUT DATASET dsCustomer):
    THIS-OBJECT:ReadData("WHERE Customer.CustNum = " + STRING(ipiCustNum)).
    RETURN CAN-FIND(FIRST ttCustomer).
END METHOD.
```

## Validation

Keep business validation in the entity rather than duplicating it in UI triggers. Return a logical result and provide an error message for the caller to display.

## UI integration

The UI obtains an entity from the factory, calls its public method, and reads or updates the returned temp-table. Database operations should remain in the entity layer.

## Database buffer rule

When directly accessing a database table, declare and use an explicit named buffer instead of the default buffer:

```abl
DEFINE BUFFER bCustomer FOR Customer.
FOR EACH bCustomer WHERE bCustomer.Country = "USA" NO-LOCK:
    /* Process the record. */
END.
```

## Testing

Test each entity operation incrementally: start with reads, then add writes and validation. Verify both expected results and error cases. Confirm dataset field definitions and data-source order match the database schema and dataset relations.
