# RAGS 3.0.60 Engine Specification

Reverse-engineered from decompiled source (ilspycmd 11.x).
Purpose: Reference for implementing/fixing rags2html's SDF parser and game engine.

---

## 1. File Format

### 1.1 Container

The `.rags` file is a **SQL Server Compact 3.5** database (`.sdf` binary format).

- **SQL CE encryption password:** `Ç°¥àòÅÅÇÉàññø`
- **Max database size:** 4024 KB
- **Connection string:** `DataSource=<path>;Max Database Size=4024;Password=Ç°¥àòÅÅÇÉàññø`

### 1.2 Application-Level Encryption (AES-256-CBC)

The `Data ntext` columns in `*Actions` tables contain AES-encrypted
SharpSerializer binary blobs (serialized `List<Action>`).

**Key derivation:**
```
password = "F1$asDDFHappy"

Step 1: RagsZone XOR transform (each char XOR 129)
Step 2: PBKDF2-SHA1, 2000 iterations
  salt = b"Ivan Medvedev"  (13 bytes: [73,118,97,110,32,77,101,100,118,101,100,101,118])
  key_material = PBKDF2(SHA1, transformed.encode('utf-16-le'), salt, 2000, 48)
  aes_key = key_material[:32]
  aes_iv  = key_material[32:48]
Step 3: AES-256-CBC
```

**Python:**
```python
import hashlib
from Crypto.Cipher import AES

def rags_zone(data: str) -> str:
    return ''.join(chr(ord(c) ^ 129) for c in data)

def derive_keys():
    pw = rags_zone("F1$asDDFHappy").encode('utf-16-le')
    km = hashlib.pbkdf2_hmac('sha1', pw, b"Ivan Medvedev", 2000, 48)
    return km[:32], km[32:48]

def decrypt_actions(ciphertext: bytes) -> bytes:
    key, iv = derive_keys()
    return AES.new(key, AES.MODE_CBC, iv).decrypt(ciphertext)
```

Note: .NET's `Rijndael.Create()` with a 32-byte key = AES-256, CBC mode (IV provided).

### 1.3 Embedded Media

The `Media` table stores raw image/video binary data in a `Data image` column.
The `.rags` file is fully self-contained — no external asset files needed.

### 1.4 Save Games

Game state is serialized via **SharpSerializer** (binary), then AES-encrypted
with the same key derivation as above.

---

## 2. Database Schema (Complete)

### 2.1 GameData (single row)

```sql
CREATE TABLE GameData(
    Title nvarchar(250),
    OpeningMessage ntext,
    HideMainPicDisplay bit,
    UseInlineImages bit,
    HidePortrait bit,
    AuthorName nvarchar(250),
    GameVersion nvarchar(50),
    GameInformation ntext,
    bgMusic nvarchar(250),
    RepeatbgMusic bit,
    PasswordProtected bit,
    GamePassword nvarchar(250),
    ObjectVersionNumber nvarchar(250),
    GameFont nvarchar(250),
    -- added by 3.0 upgrade:
    RoomGroups ntext,
    ClothingZoneLevels ntext,
    NotificationsOff bit NOT NULL DEFAULT(0)
);
```

### 2.2 Rooms

```sql
CREATE TABLE Rooms(
    UniqueID nvarchar(250),
    Description ntext,
    SDesc nvarchar(250),
    Name nvarchar(250),
    RoomPic nvarchar(250),
    EnterFirstTime bit,
    LeaveFirstTime bit,
    [Group] nvarchar(250),
    LayeredRoomPic nvarchar(250)
);
```

### 2.3 RoomExits

```sql
CREATE TABLE RoomExits (
    ID int IDENTITY(1,1) NOT NULL,
    RoomID nvarchar(250),
    Direction int,
    Active bit,
    DestinationRoom nvarchar(250),
    PortalObjectName nvarchar(250)
);
```

### 2.4 Items

```sql
CREATE TABLE Items(
    UniqueID nvarchar(250),
    Name nvarchar(250),
    Description ntext,
    SDesc nvarchar(250),
    Preposition nvarchar(250),
    LocationName nvarchar(250),
    LocationType int,
    Carryable bit,
    Wearable bit,
    Openable bit,
    Lockable bit,
    Enterable bit,
    Readable bit,
    Container bit,
    Weight float,
    Worn bit,
    [Read] bit,
    Locked bit,
    [Open] bit,
    Entered bit,
    Visible bit,
    EnterFirstTime bit,
    LeaveFirstTime bit
    -- added by 3.0 upgrade:
    -- GroupName nvarchar(255)
    -- Important bit NOT NULL DEFAULT(0)
);
```

### 2.5 Characters

```sql
CREATE TABLE Characters(
    Charname nvarchar(250),
    CharnameOverride nvarchar(250),
    CharGender int,
    CurrentRoom nvarchar(250),
    Description ntext,
    AllowInventoryInteraction bit,
    EnterFirstTime bit,
    LeaveFirstTime bit
);
```

### 2.6 Variables

```sql
CREATE TABLE Variables(
    VarName nvarchar(250),
    String ntext,
    NumType float,
    Min nvarchar(50),
    Max nvarchar(50),
    EnforceRestrictions bit,
    VarComment ntext,
    dtDateTime datetime,
    VarType int,
    VarArray ntext
);
```

### 2.7 Timer

```sql
CREATE TABLE Timer(
    Name nvarchar(250),
    TType int,
    Active bit,
    Restart bit,
    TurnNumber int,
    Length int,
    LiveTimer bit,
    TimerSeconds int
);
```

### 2.8 Player (single row)

```sql
CREATE TABLE Player(
    Name nvarchar(250),
    Description ntext,
    StartingRoom nvarchar(250),
    CurrentRoom nvarchar(250),
    PlayerLayeredImage nvarchar(250),
    PlayerGender int,
    PromptForName bit,
    PromptForGender bit,
    PlayerPortrait nvarchar(250),
    EnforceWeight bit,
    WeightLimit float
);
```

### 2.9 Media

```sql
CREATE TABLE Media(
    Name nvarchar(250),
    BackgroundColor nvarchar(250),
    TextColor nvarchar(250),
    TextFont nvarchar(250),
    ImageName nvarchar(250),
    UseEnhancedGraphics bit,
    NewImage nvarchar(250),
    Data image
);
```

### 2.10 MediaGroups

```sql
CREATE TABLE MediaGroups (
    ID int IDENTITY(1,1) NOT NULL,
    Name nvarchar(255),
    Parent nvarchar(255)
);
```

### 2.11 Child Tables (Actions & Properties)

Pattern: each entity has an `*Actions` table (encrypted command blobs) and an
`*Properties` table (key-value custom properties).

```sql
CREATE TABLE RoomActions (
    ID int IDENTITY(1,1) NOT NULL,
    RoomID nvarchar(250),
    Data ntext
);
CREATE TABLE RoomProperties (
    ID int IDENTITY(1,1) NOT NULL,
    RoomID nvarchar(250),
    Name nvarchar(250),
    Value ntext
);
CREATE TABLE ItemActions (
    ID int IDENTITY(1,1) NOT NULL,
    ItemID nvarchar(250),
    Data ntext
);
CREATE TABLE ItemProperties (
    ID int IDENTITY(1,1) NOT NULL,
    ItemID nvarchar(250),
    Name nvarchar(250),
    Value ntext
);
CREATE TABLE ItemLayeredZoneLevels (
    ID int IDENTITY(1,1) NOT NULL,
    ItemID nvarchar(250),
    Data nvarchar(250)
);
CREATE TABLE CharacterActions (
    ID int IDENTITY(1,1) NOT NULL,
    Charname nvarchar(250),
    Data ntext
);
CREATE TABLE CharacterProperties (
    ID int IDENTITY(1,1) NOT NULL,
    Charname nvarchar(250),
    Name nvarchar(250),
    Value ntext
);
CREATE TABLE VariableProperties (
    ID int IDENTITY(1,1) NOT NULL,
    VarName nvarchar(250),
    Name nvarchar(250),
    Value ntext
);
CREATE TABLE TimerActions (
    ID int IDENTITY(1,1) NOT NULL,
    Name nvarchar(250),
    Data ntext
);
CREATE TABLE TimerProperties (
    ID int IDENTITY(1,1) NOT NULL,
    TimerName nvarchar(250),
    Name nvarchar(250),
    Value ntext
);
CREATE TABLE PlayerActions (
    ID int IDENTITY(1,1) NOT NULL,
    Data ntext
);
CREATE TABLE PlayerProperties (
    ID int IDENTITY(1,1) NOT NULL,
    Name nvarchar(250),
    Value ntext
);
CREATE TABLE MediaLayeredImages (
    ID int IDENTITY(1,1) NOT NULL,
    MediaName nvarchar(250),
    Data nvarchar(250)
);
```

---

## 3. Object Model

```
Game
├── ThePlayer: Player
├── TheRooms: List<Room>
│   ├── UniqueID (GUID), Name, SDesc, Description
│   ├── RoomPic, LayeredRoomPic, [Group]
│   ├── Exits: List<Exit>
│   ├── Actions: List<Action>
│   └── CustomProperties: List<CustomProperty>
├── TheObjects: List<GameObject>  (from Items table)
│   ├── UniqueID (GUID), Name, SDesc, Description
│   ├── Preposition, LocationName, LocationType
│   ├── Carryable, Wearable, Openable, Lockable, Enterable, Readable
│   ├── Container, Weight, Worn, [Read], Locked, [Open], Entered, Visible
│   ├── EnterFirstTime, LeaveFirstTime
│   ├── Actions: List<Action>
│   └── CustomProperties: List<CustomProperty>
├── TheCharacters: List<Character>
│   ├── Charname, CharnameOverride, CharGender
│   ├── CurrentRoom, Description
│   ├── AllowInventoryInteraction, EnterFirstTime, LeaveFirstTime
│   ├── Actions: List<Action>
│   └── CustomProperties: List<CustomProperty>
├── TheVariables: List<GameVariable>
│   ├── VarName, VarType, NumType, String, dtDateTime
│   ├── Min, Max, EnforceRestrictions, VarComment
│   ├── VarArray (deserialized from ntext)
│   └── CustomProperties: List<CustomProperty>
├── TheTimers: List<Timer>
│   ├── Name, TType, Active, Restart, TurnNumber, Length, LiveTimer, TimerSeconds
│   ├── Actions: List<Action>
│   └── CustomProperties: List<CustomProperty>
├── ThePictures: List<GamePicture>  (from Media table)
│   ├── Name, BackgroundColor, TextColor, TextFont
│   ├── ImageName, UseEnhancedGraphics, NewImage
│   ├── Data (raw binary)
│   └── LayeredImages: List<string>
├── TheStatusBarItems: List<StatusBarItem>
├── QuickCommands: List<QuickMenuItem>
├── TheContent: StringBuilder
└── SortOrder: SO_NATURAL | SO_ALPHA | SO_LILO
```

---

## 4. Text Substitution Engine

### 4.1 Algorithm

```python
def perform_text_replacements(text: str, loopobject=None, newvalue: str = "") -> str:
    changed = True
    while changed:
        changed = False
        idx = text.rfind('[')  # RIGHT-TO-LEFT scan
        while idx >= 0:
            end = text.find(']', idx)
            if end == -1:
                idx = text.rfind('[', 0, idx)
                continue
            tag = text[idx:end+1].upper()
            colon = tag.find(':')
            if colon != -1:
                prefix = tag[:colon+1]
                if prefix in KNOWN_TAGS:
                    substitute(text, idx, end, prefix, loopobject, newvalue)
                    changed = True
            elif is_html_format_tag(tag):
                pass  # skip formatting tags
            else:
                # Generic [TAG] resolved against loopobject
                substitute_generic(text, idx, end, tag, loopobject)
            idx = text.rfind('[', 0, idx)
    return text
```

**Critical:** Loops until a full pass produces no changes (handles nested tokens).

### 4.2 Tag Reference

| Tag | Resolves to | Notes |
|-----|-----------|-------|
| `[V:name]` | Variable value | |
| `[RP:room:prop]` | Room custom property | `[RP:name:prop]` or `[RP:(GUID):prop]` |
| `[IP:item:prop]` | Item custom property | |
| `[PP:prop]` | Player custom property | |
| `[CP:char:prop]` | Character custom property | |
| `[VP:var:prop]` | Variable custom property | |
| `[TP:timer:prop]` | Timer custom property | |
| `[IA:item]` | Item array (loop context) | |
| `[TA:timer]` | Timer array (loop context) | |
| `[CA:char]` | Character array (loop context) | |
| `[RA:room]` | Room array (loop context) | |
| `[PA:player]` | Player array (loop context) | |
| `[JSA:var]` | Variable value, `\r`→`\\r`, `\n`→`\\n` | For JS string embedding |
| `[B]`/`[/B]` | `<b>`/`</b>` | |
| `[I]`/`[/I]` | `<i>`/`</i>` | |
| `[U]`/`[/U]` | `<u>`/`</u>` | |
| `[F color]` | Font color | |
| `[C color]` | Color (alt) | |
| `[MIDDLE]` | Vertical center | |

### 4.3 Built-in JavaScript

Injected into the HTML content area on load:
```javascript
function getVerticalScrollPosition() { return document.body.scrollTop.toString(); }
function setVerticalScrollPosition(position) { document.body.scrollTop = position; }
function pagedown() { document.body.scrollTop += document.body.clientHeight; }
function atbottom() {
    return (document.body.scrollTop + document.body.clientHeight) >= document.body.scrollHeight;
}
```

---

## 5. Command System

### 5.1 Structure

| Field | Type | Purpose |
|-------|------|---------|
| `cmdtype` | CommandType enum | Action to perform |
| `CommandText` | string | Primary param (SQL query or variable name) |
| `CommandPart2` | string | Secondary param |
| `CommandPart3` | string | Tertiary param |
| `CommandPart4` | string | Quaternary param |
| `CustomChoices` | List\<string\> | For input commands |
| `EnhInputData` | EnhancedInputData | Enhanced input dialog config |

All text fields pass through `PerformTextReplacements` before use.

### 5.2 CT_Execute_SQL

```
text  = PerformTextReplacements(CommandPart2)  → target variable name
text4 = PerformTextReplacements(CommandText)   → SQL query
→ Execute query → store result rows into variable.VarArray as List<List<string>>
```

### 5.3 Full CommandType Enum (111 entries)

```
ct_uninitialized
ct_action_add_customchoice
ct_action_clear_customchoice
ct_action_remove_customchoice
ct_displaychardesc
ct_char_displayport
ct_movechar
ct_movecharinvtoplayer
ct_char_movetoobj
ct_char_setport
ct_setcharaction
ct_char_set_custom_property
ct_char_set_custom_property_js        ← 3.0
ct_setchardesc
ct_char_set_gender
ct_char_set_name
ct_comment
ct_debugtext
ct_java_set_rags                      ← 3.0 (js execution)
ct_displaytext
ct_exportvariable
ct_importvariable
ct_pausegame
ct_displaylayeredpicture
ct_displaypicture
ct_mm_set_background_music            ← 3.0
ct_mm_stop_background_music           ← 3.0
ct_mm_play_soundeffect                ← 3.0
ct_mm_set_main_compass                ← 3.0
ct_mm_set_ud_compass                  ← 3.0
ct_layeredimage_add                   ← 3.0
ct_layeredimage_clear                 ← 3.0
ct_layeredimage_remove                ← 3.0
ct_layeredimage_replace               ← 3.0
ct_displayitemdesc
ct_item_layered_remove                ← 3.0
ct_item_layered_wear                  ← 3.0
ct_moveitemtochar
ct_moveitemtoinv
ct_moveitemtoobj
ct_moveitemtoroom
ct_setobjectaction
ct_item_set_custom_property
ct_item_set_custom_property_js        ← 3.0
ct_setitemdesc
ct_item_set_name_override
ct_setlockedunlocked
ct_setopenclosed
ct_setitemtoworn
ct_item_set_visibility
ct_items_move_container_to_container
ct_loop_break
ct_cancelmove
ct_displayplayerdesc
ct_setlayeredplayerportrait           ← 3.0
ct_moveinventorytochar
ct_moveinventorytoroom
ct_moveplayer
ct_movetochar
ct_movetoobj
ct_player_set_custom_property
ct_player_set_custom_property_js      ← 3.0
ct_setplayeraction
ct_setplayerdesc
ct_setplayergender
ct_setplayername
ct_setplayerportrait
ct_displayroomdescription
ct_displayroompicture
ct_room_move_items_to_player
ct_setroomaction
ct_room_set_custom_property
ct_room_set_custom_property_js        ← 3.0
ct_setroomdescription
ct_setexit
ct_setexitdestination
ct_room_set_name_override
ct_setroomlayeredpic                  ← 3.0
ct_setroompic
ct_execute_sql
ct_status_itemvisibleinvisible
ct_executetimer
ct_resettimer
ct_timer_set_custom_property
ct_timer_set_custom_property_js       ← 3.0
ct_settimer
ct_displayvariable
ct_setvariable
ct_item_getrandomgroup
ct_mm_getrandomgroup                  ← 3.0
ct_variable_set_javascript            ← 3.0
ct_setvariablebyinput
ct_setvariable_numeric_byinput
ct_variable_set_custom_property
ct_variable_set_custom_property_js    ← 3.0
ct_variable_set_randomly
ct_variable_set_with_charpropertyvalue
ct_variable_set_with_itempropertyvalue
ct_variable_set_with_playerpropertyvalue
ct_variable_set_with_roompropertyvalue
ct_variable_set_with_timerpropertyvalue
ct_variable_set_with_variable
ct_variable_set_with_variablepropertyvalue
ct_endgame
```

---

## 6. Condition System

### 6.1 Structure

```
Condition
├── conditionname: string
├── Checks: List<Check>
├── PassCommands: List<Command>
└── FailCommands: List<Command>

Check
├── CondType: ConditionType (38 values)
├── CkType: CheckType (And | Or)  ← how this check combines with others in the Condition
├── ConditionStep2: string  (resolved via PerformTextReplacements)
├── ConditionStep3: string
└── ConditionStep4: string  (often the comparison operator, e.g. ">", "=", "Contains")
```

### 6.2 CheckType (logical combinator)

```
CT_Uninitialized
And   → all Checks in the Condition must pass
Or    → any Check in the Condition may pass
```

### 6.3 ConditionType → Step Interpretation

| CondType | Step2 | Step3 | Step4 |
|----------|-------|-------|-------|
| CT_Item_Held_By_Character | Character name | Item name | — |
| CT_Item_Held_By_Player | — | Item name | — |
| CT_Item_In_Object | Object name | Item name | — |
| CT_Item_In_Room | Room name | Item name | — |
| CT_Player_In_Room | Room name | — | — |
| CT_Player_In_Same_Room_As | Character name | — | — |
| CT_Variable_Comparison | Variable name | Value | Operator (=, >, <, Contains, etc.) |
| CT_Variable_To_Variable_Comparison | Var 1 name | Var 2 name | Operator |
| CT_*_CustomPropertyCheck | Entity name | Property name | Expected value / Operator |

### 6.4 Loop Conditions

`CT_Loop_*` iterate a collection, passing each element as `loopobject` to all
nested `PerformTextReplacements` calls.

---

## 7. GameVariable

```
GameVariable
├── VarName: string
├── VarType: VT_NUMBER | VT_STRING | VT_DATETIME | VT_NUMBERARRAY | VT_STRINGARRAY | VT_DATETIMEARRAY
├── NumType: double
├── String: string
├── dtDateTime: DateTime
├── Min: string (nvarchar(50), may contain [V:] tags)
├── Max: string (nvarchar(50))
├── EnforceRestrictions: bool
├── VarComment: string
├── VarArray: deserialized from ntext
└── CustomProperties: List<CustomProperty>
```

---

## 8. Assembly Layout

| Assembly | Namespace | Role |
|----------|-----------|------|
| `Rags.exe` | RAGSWPFTEST1 | WPF shell, command dispatch switch |
| `RagsSvPlayerLib.dll` | Rags | Core engine: Game, Command, Condition, Check, GameObject, PerformTextReplacements |
| `WPFRagsLib.dll` | WPFRagsLib, Rags | SaveLoad (SDF I/O), EncDec (AES), WPF controls |
| `RagsDesigner.exe` | WPFRagsDesigner | Designer (not needed for playback) |

---

## 9. Known Gaps in rags2html

| Feature | Severity | Notes |
|---------|----------|-------|
| `CT_VARIABLE_SET_JAVASCRIPT` | **High** | Needs JS sandbox or explicit no-op+warning |
| `CT_JAVA_SET_RAGS` | **High** | Same |
| `CT_*_SET_CUSTOM_PROPERTY_JS` (×6) | **High** | Same |
| `CT_LAYEREDIMAGE_*` (×4) | Medium | Layered image compositing |
| `CT_MM_*` (×5) | Medium | Music/sound |
| `CT_SETROOMLAYEREDPIC` | Medium | |
| `CT_SETLAYEREDPLAYERPORTRAIT` | Medium | |
| `CT_ITEM_LAYERED_WEAR/REMOVE` | Medium | |
| `CT_VARIABLE_SET_WITH_*PROPERTYVALUE` (×6) | Medium | Read property → variable |
| `CT_ITEM_GETRANDOMGROUP` / `CT_MM_GETRANDOMGROUP` | Low | |
| `CT_EXPORTVARIABLE` / `CT_IMPORTVARIABLE` | Low | |
| `CT_SETVARIABLEBYINPUT` / `CT_SETVARIABLE_NUMERIC_BYINPUT` | Low | |
| `CT_VARIABLE_SET_RANDOMLY` | Low | |
| `CT_PAUSEGAME` / `CT_CANCELMOVE` / `CT_LOOP_BREAK` | Low | |
| `CT_ENDGAME` | Low | |
| `CT_COMMENT` / `CT_DEBUGTEXT` | None | No-op / debug |

---

## 10. Quick Reference: Opening a .rags File in Python

```python
import hashlib
from Crypto.Cipher import AES

SQLCE_PASSWORD = "Ç°¥àòÅÅÇÉàññø"
AES_PASSWORD = "F1$asDDFHappy"
SALT = b"Ivan Medvedev"
XOR_KEY = 129

def rags_zone(data: str) -> str:
    return ''.join(chr(ord(c) ^ XOR_KEY) for c in data)

def derive_keys():
    pw = rags_zone(AES_PASSWORD).encode('utf-16-le')
    km = hashlib.pbkdf2_hmac('sha1', pw, SALT, 2000, 48)
    return km[:32], km[32:48]

def decrypt_actions(ciphertext: bytes) -> bytes:
    key, iv = derive_keys()
    return AES.new(key, AES.MODE_CBC, iv).decrypt(ciphertext)

# SDF file access: use pysqlce or parse binary format directly (see rags2html)
# The file-level encryption uses SQLCE_PASSWORD.
# The application-level AES (above) is for *Actions Data columns only.
```

---

## 11. Sources

All from decompilation of RAGS Suite 3.0.60:
- `Rags.exe` → `RAGSWPFTEST1/MainWindow.xaml.cs`
- `RagsSvPlayerLib.dll` → `Rags/Game.cs`, `Rags/Command.cs`, `Rags/Condition.cs`, `Rags/Check.cs`, `Rags/CheckType.cs`, `Rags/GameVariable.cs`, `Rags/GameObject.cs`
- `WPFRagsLib.dll` → `WPFRagsLib/SaveLoad.cs`, `Rags/EncDec.cs`
