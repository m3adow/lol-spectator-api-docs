# REST Service for LoL spectators
This is an unofficial, uncomplete and (pretty sure) wrong documentation of the RESTful service which powers the League of Legends spectator mode. 

This documentation is desgined to be community driven and should be extended by everyone. If you find things missing, add them please!

## How it works

Riot's spectator mode works by requesting replay data via HTTP form a service. 
The data is split in chunks which usually contain about 30 seconds of gameplay. Additionally there are key frames which seem to contain more information than a single chunk. They seem to be used to support 
skipping back in time or for spectators that join the game later. 
This means that the key frames should contain every information about 
the state of a game at a given point in time. On the other hand, chunks may only contain changes to this state. 

This leads to the following process for replaying a series of chunks:

1. Get server version
2. Get metadata of the game
3. Get all the Startup Chunks (until the endStartupChunkId)
4. Read the last available key frame
5. Read chunk following this key frame
6. Read next chunk
7. Repeat 4, 5, 6 until end of game or skip

The same procedure is known to work in many video and audio formats. It provides a fast and bandwidth saving way of deliviering all information fast while it also gives a good way to jump quickly to any point in time without the need to replay every chunk from the start to that moment. 

## Cryptography

Chunks and Keyframes are encrypted and compressed. The algorithm used for cryptography seems to be Blowfish in ECB mode. Compression is done using zip BEFORE the encryption. So to read a chunks data one needs to first decrypt then uncompress the data. 

## API Docs

### URL Template

Most of the REST URLs look like this: 
```
http://<REST host:port>/observer-mode/rest/consumer/<method>/<platformId>/<gameID>/<parameter>/token'
``` 
**Parameters**

* **host:port**: there are currently 8 REST services running for the different regions. A list of these servers exists at the end of this document. 
* **method**: Name of the method you want to call. See "*Methods*".
* **platformId**/**gameID**: Every game is uniquely identified by the short name of its region and a numeric game ID. 
* **parameter**: The methods first and single parameter (aside of gameid/platformId). If a method doesn't require an additional parameter this value is '1'

### Methods

#### featured() > application/json

URL **.../observer-mode/rest/featured**

Lists the 10 featured games for the regions supported by this server. 

#### version() > text/plain

URL: **.../observer-mode/rest/consumer/version**

Contains the current version for this Region.  

#### getGameMetaData(platformId, gameId) > application/json
URL: **.../observer-mode/rest/consumer/getGameMetaData/&lt;platformId&gt;/&lt;gameID&gt;/1/token**

Returns information about the given game :

* **gameKey**
  * **gameId**: The unique game ID
  * **platformId**: The ID of the current server (EUW1, ...)
* **gameServerAddress**: Empty field
* **port**: equal to 0
* **encryptionKey**: Empty field
* **chunkTimeInterval**: The average time that a chunk represents, usually equal to 30000 (ms)
* **startTime**: The game start time as a string object (ex: "Apr 21, 2016 1:38:10 PM")
* **gameEnded**: Set to true if the game ended
* **lastChunkId**: Id of the last available chunk
* **lastKeyFrameId**: Id of the last available key frame
* **endStartupChunkId**: Id of the last startup chunks
* **delayTime**: Unknown (?) Usually set to 150000
* **pendingAvailableChunkInfo**: Contains an array of the available chunks
* **pendingAvailableKeyFrameInfo**: Contains an array of the available key frames
* **keyFrameTimeInterval**: The average time between two key frames, usually set to 60000 (ms), 1 keyframe every 2 chunks
* **decodedEncryptionKey**: Empty field
* **startGameChunkId**: Id of the first chunk since the game really start
* **gameLength**: The game length in s (?), set to 0 if the game is in progress
* **clientAddedLag**: Lag added by the client, usually set to 30000 (ms)
* **clientBackFetchingEnabled**: Unknown (?), usually set to false
* **clientBackFetchingFreq**: Unknown (?), usually set to 1000
* **interestScore**: Unknown (?), usually near 2000
* **featuredGame**: Set to true if the current game is a featured one
* **createTime**: The game creation time as a string object (ex: "Apr 21, 2016 1:38:10 PM")
* **endGameChunkId**: Id of the last chunk of the game. Set to -1 if the game is still in progress
* **endGameKeyFrameId**: Id of the last key frame of the game. Set to -1 if the game is still in progress

#### getLastChunkInfo(platformId, gameId) > application/json
URL: **.../observer-mode/rest/consumer/getLastChunkInfo/&lt;platformId&gt;/&lt;gameID&gt;/1/token**

Return some information about the last avaiable chunk: 

* **chunkId**: ID of the last available chunk,
* **availableSince**: Number of ms this chunk is available,
* **nextAvailableChunk**: Number of ms until the next chunk is available,
* **keyFrameId**: key frame that belongs to this chunk,
* **nextChunkId**: chunk that directly follows this key frame,
* **endStartupChunkId**: chunk that determinates the end of pick&ban phase,
* **startGameChunkId**: first chunk of the actual game,
* **endGameChunkId**: last chunk of the actual game (or 0 if game still running),
* **duration**: Number of ms this chunks information represents

#### endOfGameStats(platformId, gameId) > base64 encoded AMF Data
URL: **.../observer-mode/rest/consumer/endOfGameStats/&lt;platformId&gt;/&lt;gameID&gt;/null**

Contains data used for the statistics screen after a game.

Example Content: http://pastebin.com/KB4TUPhs (thanks to [Divi](https://github.com/Divi))

#### getGameDataChunk(platformId, gameId, chunkId) > ? Encrypted, Compressed
URL: **.../observer-mode/rest/consumer/getGameDataChunk/&lt;platformId&gt;/&lt;gameID&gt;/&lt;chunkId&gt;/token**

Retrieves a chunk of data for the given game. 

#### getKeyFrame(platformId, gameId, keyFrameId) > ? Encrypted, Compressed
URL: **.../observer-mode/rest/consumer/getKeyFrame/&lt;platformId&gt;/&lt;gameID&gt;/&lt;keyFrameId&gt;/token**

Retrieves a key frame for the given game.

## Servers
Note: The time zones indicate which time zone that server will send date properties in. Initially determined by aj-r. The times may have changed since.

| Name                 | Region | PlatformID | Domain                                | Time Zone |
|----------------------|--------|------------|---------------------------------------|-----------|
| North America        | NA     | NA1        | spectator.na.lol.riotgames.com:80     | Pacific   |
| Europe West          | EUW    | EUW1       | spectator.euw1.lol.riotgames.com:80   | UTC       |
| Europe Nordic & East | EUNE   | EUN1       | spectator.eu.lol.riotgames.com:8088   | Pacific   |
| Japan                | JP     | JP1        | spectator.jp1.lol.riotgames.com:80    | *Untested*|
| Brazil               | BR     | BR1        | spectator.br.lol.riotgames.com:80     | Pacific   |
| Latin America North  | LAN    | LA1        | spectator.la1.lol.riotgames.com:80    | Pacific   |
| Latin America South  | LAS    | LA2        | spectator.la2.lol.riotgames.com:80    | Pacific   |
| Russia               | RU     | RU         | spectator.ru.lol.riotgames.com:80     | Pacific   |
| Turkey               | TR     | TR1        | spectator.tr.lol.riotgames.com:80     | Pacific   |
| Public Beta Env.     | PBE    |PBE1        | spectator.pbe1.lol.riotgames.com:8088 | Pacific   |

[Official Source](https://developer.riotgames.com/docs/spectating-games)

## Sources
* https://github.com/robertabcd/lol-ob/issues/2#issuecomment-25749662
* https://gist.github.com/lukegb/d2997a5fc7970ce6e1e1
* https://github.com/robertabcd/lol-ob/wiki/ROFL-Container-Notes
* https://github.com/robertabcd/lol-ob/blob/master/rofl.rb
* https://gist.github.com/themasch/8375971
* https://gist.github.com/aj-r/788b9abcde6afdf527c1
* https://gist.github.com/Aztorius/e428be6515b19fd24823754b72038e1b
