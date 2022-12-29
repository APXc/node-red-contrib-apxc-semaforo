# SEMAFORO Global Multi-Instance Backup
Node that manages an object or lists of objects,
with values to be used and varied in a
transfers to the individual stream, 
all while maintaining a backup of values in case
of blockage/recovery

## General Properties
 -  **Stoplight**
     Traffic Light Monitoring Status 
 - **Road**
    Reference Topic for the current instance
 - **TypeReq**
     the type of request used
 - **PropsType**
     Gesite Object Type, SingleValue(a Single Object), ListMultiObject(a multi Object with similar structure)
 - **NamePropsID** 
     name of the Property identifying the object/value to be updated
 - **NameAdditionaListProp** 
     Object with Additional Properties to Update
 - **BackupName** 
     Name Assigned to the backup instance
 - **Overwrite**
     Overwriting of current values

### Stoplight
 A property that defines whether the Traffic Light will monitor the status (Cycle Closed) or ignore it (Cycle Open and Infinite) 

### Road 
 sting with which to identify the context to be used
 for the values, so that even more instances with different objects to be monitored can be maintained

### TypeReq
 The Type of Request can have the following values::
 - **INIT** - Initiator, compulsory in the instance flow to enable proper functioning of the whole
 - **GET** - get of current values
 - **UPD** - Updating Data, passing the property with bulean value `msg.status` update the status
 - **RESET** - Elimination of the BK's Ongoing Status - recommended to be used only for cleaning error situations
 - **GET BACKUP** - Returns the contents of the BACKUP file 

### PropsType
  the type of Object to be managed, may have the following values:
 - **SingleObject** - Single object with a single status to be monitored
 - **ListMultiObject** - list of multi-key objects with multiple staus to be monitored

#### _Returned SingleObject Exsample_
   `singleValue` : `{"value":{"name":"Alfa","time":30, "test": true},"status":false}` 

#### _Returned ListMultiObject Exsample_
   `tableList`: `[{"name":"alfa","time":30,"status":true},{"name":"beta","time":56,"status":false}]` 

### NamePropsID
 _(Required for ListMultiObject on UPD)_ name of key in `msg` which contains the value to identify in the object list the element to be updated

### NameAdditionaListProp
_(only usable in UPD)_ nname of key in `msg` which contains an object properties and values, other than the status to be updated in the monitored element

### BackupName
 string to be used as the name of the current Road backup file

### Overwrite 
 _(only usable in INIT)_ property that allows in Initialisation to ignore the backup and currently monitored objects to create a clean initial situation

---

## Object Property
 - **`OnReq`** - Type: `Bool` - It indicates the fact that at least one of the values and with `status` false,
 - **`singleValue`** - Type: `Object` - contains within the key property `value` he value of the single object to be monitored passed in the initial or backup and a in the key `status` contains the state of the object that determines current `OnReq` ,
 - **`tableList`** - Type: `Array<Object>` - contains the list of objects passed in the initialisation with the addition in each object of the key `status` which contains the state of the object, the sum of the states of all objects determines current `OnReq`
---

# Flow instance 
the application flow provides:
 1. an **INIT** block to initialize values/retrieve the backup, lock the SEMAFO in `OnReq` state to `true` and initialise the status values of the monitored objects to `false
 2. a **GET** block to get the current state data 
 3. a **UPD** block to update the state and/or property values of the items
 4. if needed a **RESET** block to clean up the values

the stream is "_Unlocked_" (i.e. in `OnReq` to `false`) when the state of all its elements is `true`.

---

# _Returned Object Exsample_

`msg.payload` : `{"onReq":true,"singleValue":{"value":{"name":"Alfa","time":30, "test": true},"status":false},"tableList":[{"name":"alfa","time":30,"status":true},{"name":"beta","time":56,"status":false}]}`

---
# Backup Logic
the node generates for each **Road** a backUp file named as specified in `BackupName`,
this file contains the history of updates to the `singleValue` and `tableList` objects,
all as long as the `OnReq` key remains in the `true` state, when it changes state the saved values will be cleared to allow a new cycle,
the file is used to retrieve the values and states of the monitored objects, so in case of a new start the values will not be overwritten (if the **Overwrite** property is not set) and in case the flow breaks down unexpectedly the values can be retrieved

# Closed-Cycle Vision
the underlying logic of the node is to make the status recoverable even in the event of an engine crash, 
as a consequence, the change data of the monitored objects will be kept in the back-up file until the click is complete,
in the event of a block at the new start, if the click had been completed, a new one will be injected, 
Otherwise, the node will ignore the passed values and retrieve the data from the back-up file.

# Backup File
the contents of the backup file are as follows:
 - **`name`** - Type: `String` - Name of files,
 - **`stoplight`** - Type: `Bool` - Monitor Status Traffic light for instance under backup,
 - **`completed`** - Type: `Bool` - Indicates the status of the current cycle,
 - **`createdAt`** - Type: `DataTime` - file creation date,
 - **`road`** - Type: `String`- **Road** of affiliation,
 - **`history`** - Type: `Array<Object>` - contains the list of the various versions of the monitor objects, tracking the parameters passed and date of update

# _Returned Object Exsample_

`msg.payload`: `{"name":"TestList", "stoplight":true,"completed":false,"createdAt":"2022-11-30T16:11:50.531Z","road":"ALFA_ONE","history":[{"onReq":true,"singleValue":null,"tableList":[{"name":"alfa","time":"TR","status":false},{"name":"beta","time":"TH","status":false}],"file":"BK_GBL_PROPS/ALFA_ONE_TestList.json","version":0,"datetime":"2022-12-05T09:37:53.648Z","propGetRif":"name"},{"onReq":true,"singleValue":null,"tableList":[{"name":"alfa","time":"TR","status":false},{"name":"beta","time":"PROVE","status":true}],"file":"BK_GBL_PROPS/ALFA_ONE_TestList.json","version":1,"datetime":"2022-12-05T09:38:06.798Z"}]}`
