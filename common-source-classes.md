---
title: "Common Telegram Classes"
source: "https://plugins.exteragram.app/docs/common-source-classes"
---

# Common Telegram Classes

Reference for frequently used Telegram source classes and their locations for plugin development.

## Links to commonly used Telegram classes

It is recommended to have a local copy of Telegram sources, opened in Android Studio.

### LaunchActivity

[View on GitHub](<https://github.com/DrKLO/Telegram/blob/master/TMessagesProj/src/main/java/org/telegram/ui/LaunchActivity.java>)

Path: `TMessagesProj/src/main/java/org/telegram/ui/LaunchActivity.java`

That's where app initialization happens. Custom links are handled here as well.

### ProfileActivity

[View on GitHub](<https://github.com/DrKLO/Telegram/blob/master/TMessagesProj/src/main/java/org/telegram/ui/ProfileActivity.java>)

Path: `TMessagesProj/src/main/java/org/telegram/ui/ProfileActivity.java`

User and channel profile fragment.

### ChatActivity

[View on GitHub](<https://github.com/DrKLO/Telegram/blob/master/TMessagesProj/src/main/java/org/telegram/ui/ChatActivity.java>)

Path: `TMessagesProj/src/main/java/org/telegram/ui/ChatActivity.java`

Chat rendering and functionality.

### MessageObject

[View on GitHub](<https://github.com/DrKLO/Telegram/blob/master/TMessagesProj/src/main/java/org/telegram/messenger/MessageObject.java>)

Path: `TMessagesProj/src/main/java/org/telegram/messenger/MessageObject.java`

Wrapper for `TLRPC.Message`.

### ChatMessageCell

[View on GitHub](<https://github.com/DrKLO/Telegram/blob/master/TMessagesProj/src/main/java/org/telegram/ui/Cells/ChatMessageCell.java>)

Path: `TMessagesProj/src/main/java/org/telegram/ui/Cells/ChatMessageCell.java`

Handles message rendering.

### AndroidUtilities

[View on GitHub](<https://github.com/DrKLO/Telegram/blob/master/TMessagesProj/src/main/java/org/telegram/messenger/AndroidUtilities.java>)

Path: `TMessagesProj/src/main/java/org/telegram/messenger/AndroidUtilities.java`

Contains a lot of useful utilities.

### MessagesController

[View on GitHub](<https://github.com/DrKLO/Telegram/blob/master/TMessagesProj/src/main/java/org/telegram/messenger/MessagesController.java>)

Path: `TMessagesProj/src/main/java/org/telegram/messenger/MessagesController.java`

Contains all methods related to managing app state and Telegram requests.

### MessagesStorage

[View on GitHub](<https://github.com/DrKLO/Telegram/blob/master/TMessagesProj/src/main/java/org/telegram/messenger/MessagesStorage.java>)

Path: `TMessagesProj/src/main/java/org/telegram/messenger/MessagesStorage.java`

Manages local database state. You may use `database` field to execute custom SQLite queries.

### SendMessagesHelper

[View on GitHub](<https://github.com/DrKLO/Telegram/blob/master/TMessagesProj/src/main/java/org/telegram/messenger/SendMessagesHelper.java>)

Path: `TMessagesProj/src/main/java/org/telegram/messenger/SendMessagesHelper.java`

Contains methods for sending all kind of messages, including files.

### BulletinFactory

[View on GitHub](<https://github.com/DrKLO/Telegram/blob/master/TMessagesProj/src/main/java/org/telegram/ui/Components/BulletinFactory.java>)

Path: `TMessagesProj/src/main/java/org/telegram/ui/Components/BulletinFactory.java`

Bulletins are small notification messages displayed on the bottom.

### AlertDialog

[View on GitHub](<https://github.com/DrKLO/Telegram/blob/master/TMessagesProj/src/main/java/org/telegram/ui/ActionBar/AlertDialog.java>)

Path: `TMessagesProj/src/main/java/org/telegram/ui/ActionBar/AlertDialog.java`

Alert dialogs are shown on top of current fragment. They support custom handlers on action buttons.

### TLRPC (all Telegram request models)

Path: `TMessagesProj/src/main/java/org/telegram/tgnet`

  * [TLRPC.java](<https://github.com/DrKLO/Telegram/blob/master/TMessagesProj/src/main/java/org/telegram/tgnet/TLRPC.java>)
  * [Additional models](<https://github.com/DrKLO/Telegram/tree/master/TMessagesProj/src/main/java/org/telegram/tgnet/tl>)

Human-readable list: <https://corefork.telegram.org/schema> (not always up-to-date)

[PreviousAvailable Libraries](</docs/available-libraries>)
