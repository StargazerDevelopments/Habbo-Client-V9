# Habbo Client V9

Custom Habbo v9 dcrs with improved code, new features and more!

**NOTE:**<br>
Please note that this requires a custom v9 server. Normal v9 servers will be broken due to invalid packets. Check my Github for a custom fork of Kepler working on these DCRs. 

**Glossary:**<br>
S > C: Server > Client, all packets sent by the server and handled by the client<br>
C > S: Client > Server, all packets sent by the client and handled by the server (session)

**TODO**:
- General changes + new features
    - Create a separate Habbo.dir/fuse_client.cst for scaled client
    - Add SSO ticket login
    - Add windows
    - Add landscape
    - Add infofeed from v28+
    - Find more cool features
- hh_cat_code
    - Change `on handle_purchase_error` to read an integer from request data for error code
    - Change `on handle_purchase_nobalance` to not read any unnecessary request data
    - Change `on handle_catalogindex` to the new packet format (S > C)
    - Change `on handle_catalogpage` to the new packet format (S > C)
    - Change `on purchaseProduct` to the new packet format (C > S)
    - Change `on retrieveCatalogueIndex` to the new packet format (C > S)
    - Change `on retrieveCataloguePage` to the new packet format (C > S)
- hh_entry
    - Remove a check in `on handleSecretKey` so it doesn't empty `tClientUrl` if the `tHost` is containing a specific string (which is obfuscated but IIRC is localhost)
    - Change `on handleUserObj` to the new packet format (S > C)
    - Change `on handleEPSnotify` to the new packet format (S > C)
    - Change `on handleErr` to the new packet format (S > C)
    - Remove `GET_PASSWORD` from `tCmds` or use it since it is never sent
- hh_kiosk_room
    - Change `on handle_flatcreated` to the new packet format (S > C)
    - Change `on sendNewRoomData` to the new packet format (C > S)
    - Change `on sendSetFlatInfo` to the new packet format (C > S)
- hh_messenger
    - Change `on get_buddy_info` so `#id` is read as an integer instead as a string converted to an integer
    - Change `on get_buddy_info` so `#name` doesn't do an unnecessary conversion from string to string
    - Change `on get_user_info` so `#id` is read as an integer instead as a string converted to an integer
    - Change `on get_console_message` so `#id` and `senderID` are read as an integer instead as a string converted to an integer
    - Change `on get_campaign_message` so `#id` is read as an integer instead as a string converted to an integer
    - Change `on get_buddy_request` so `#id` is read as an integer instead as a string converted to an integer
- hh_navigator
    - Change `on handle_flat_results` to the new packet format (S > C)
    - Remove `SBUSYF` FROM `tCmds` or use it since it is never sent
    - Change `on sendGetOwnFlats` so it doesn't append the logged in username to the SUSERF packet
    - Change `on sendGetFlatInfo` so the flat id gets sent as an integer
    - Change `on sendDeleteFlat` so the flat id gets sent as an integer
    - Change `on sendupdateFlatInfo` to the new packet format (C > S)
- hh_photo:
    - To check out if `on handle_film_mus` needs to be changed since at the moment I'm not sure what it is
- hh_purse:
    - Change `on_handle_purse` so the amount of credits is read as an integer, instead of as a string, parsed as a float converted to an integer
    - Change `on_handle_purse` to the new packet format (S > C)
    - Clean up `on handle_purse` so every `tMsg.subject` goes in their own handle function
    - Change `on handle_tickets` so it reads the ticket amount as int
    - Change `on handle_ticketsbuy` so it reads the ticket amount as int
- hh_registrat
    - Change `on sendNewFigureDataToServer` so the `REGISTER` packet is sent better to the server to read easier
    - Change `on sendFigureUpdateToServer` so the `UPDATE` packet is sent better to the server to read easier
    - Change `on sendUpdateAccountMsg` so the `UDPATE_ACCOUNT` packet is sent better to the server to read easier
- hh_room
    - Admittedly I got bored here and hh_room is big so I decided not to do this.. yet. Sorry okay, this stuff is boring :(

# New packet format (S > C)
Habbo changed their packet format. In older versions, it was often done by reading the amount of lines and splitting data with tabs. For example:

```
on handle_flat_results me, tMsg
  tResult = [:]
  tList = [:]
  tDelim = the itemDelimiter
  the itemDelimiter = TAB
  repeat with i = 1 to tMsg.content.line.count
    tLine = tMsg.content.line[i]
    if (tLine = EMPTY) then
      exit repeat
    end if
    tFlat = [:]
    tFlat[#id] = ("f_" & tLine.item[1])
    tFlat[#flatId] = tLine.item[1]
    tFlat[#name] = tLine.item[2]
    tFlat[#owner] = tLine.item[3]
    tFlat[#door] = tLine.item[4]
    tFlat[#port] = tLine.item[5]
    tFlat[#usercount] = tLine.item[6]
    tFlat[#maxUsers] = tLine.item[7]
    tFlat[#Filter] = tLine.item[8]
    tFlat[#description] = tLine.item[9]
    tFlat[#nodeType] = 2
    tList[tFlat[#id]] = tFlat
  end repeat
  tResult.addProp(#children, tList)
  case tMsg.subject of
    16:
      tResult[#id] = #own
    55:
      tResult[#id] = #src
  end case
  the itemDelimiter = tDelim
  me.getComponent().saveFlatResults(tResult)
end
```

You can see it goes through all lines and uses the TAB as delimiter for each row. Now lets take a look at the same packet handler in v39:

```
on parseFlatResults me, tMsg
  tConn = tMsg.getaProp(#connection)
  if not tConn then
    return 0
  end if
  tList = [:]
  tFlatCount = tConn.GetIntFrom()
  repeat with i = 1 to tFlatCount
    tFlat = [:]
    tID = tConn.GetIntFrom()
    tFlat[#id] = ("f_" & tID)
    tFlat[#flatId] = tID
    tFlat[#name] = tConn.GetStrFrom()
    tFlat[#owner] = tConn.GetStrFrom()
    tFlat[#door] = tConn.GetStrFrom()
    tFlat[#usercount] = tConn.GetIntFrom()
    tFlat[#maxUsers] = tConn.GetIntFrom()
    tFlat[#description] = tConn.GetStrFrom()
    tFlat[#nodeType] = 2
    tList[tFlat[#id]] = tFlat
  end repeat
  return tList
end
```

Here it reads an int, loops that amount of times through the data and simply reads data as either wire/VL64 encoded integer or string ending with a char(2) break. This format is cleaner and easier to work with so this is the format that should be used.

# New packet format (c > S)
Just like with the S > C, Habbo changed some packets when sending to client.

For example:

```
on purchaseProduct me, tGiftProps
  if (pProductOrderData.ilk <> #propList) then
    return error(me, "Incorrect Product data", #purchaseProduct)
  end if
  if (tGiftProps.ilk <> #propList) then
    return error(me, "Incorrect Gift Props", #purchaseProduct)
  end if
  if voidp(pProductOrderData["name"]) then
    return error(me, "Product name not found", #purchaseProduct)
  end if
  if voidp(pProductOrderData["purchaseCode"]) then
    return error(me, "PurchaseCode name not found", #purchaseProduct)
  end if
  if voidp(pProductOrderData["extra_parm"]) then
    pProductOrderData["extra_parm"] = "-"
  end if
  if voidp(pCatalogProps["editmode"]) then
    return error(me, "Catalogue mode not found", #purchaseProduct)
  end if
  if voidp(pCatalogProps["lastPageID"]) then
    return error(me, "Catalogue page id missing", #purchaseProduct)
  end if
  if not voidp(tGiftProps["gift"]) then
    tGift = (tGiftProps["gift"] & RETURN)
    if not voidp(tGiftProps["gift_receiver"]) then
      tGift = ((tGift & tGiftProps["gift_receiver"]) & RETURN)
    else
      tGift = EMPTY
    end if
    if not voidp(tGiftProps["gift_msg"]) then
      tGiftMsg = tGiftProps["gift_msg"]
      tGiftMsg = convertSpecialChars(tGiftMsg, 1)
      tGift = ((tGift & tGiftMsg) & RETURN)
    else
      tGift = EMPTY
    end if
  else
    tGift = "0"
  end if
  tOrderStr = EMPTY
  tOrderStr = ((tOrderStr & pCatalogProps["editmode"]) & RETURN)
  tOrderStr = ((tOrderStr & pCatalogProps["lastPageID"]) & RETURN)
  tOrderStr = ((tOrderStr & me.getLanguage()) & RETURN)
  tOrderStr = ((tOrderStr & pProductOrderData["purchaseCode"]) & RETURN)
  tExtra = pProductOrderData["extra_parm"]
  tExtra = convertSpecialChars(tExtra, 1)
  tOrderStr = ((tOrderStr & tExtra) & RETURN)
  tOrderStr = (tOrderStr & tGift)
  if not connectionExists(getVariable("connection.info.id")) then
    return 0
  end if
  return getConnection(getVariable("connection.info.id")).send("GPRC", tOrderStr)
end
```

Here you need to split the packet in lines. Then, if something isn't an integer, it needs to be parsed to int. This method is more prone to mistakes. Now, here's the same packet (different name, but same header) in v39

```
on sendPurchaseFromCatalog me, tPageID, tOfferCode, tExtraParam, tAsGift, tGiftReceiver, tGiftMessage
  tMsg = [:]
  tMsg.addProp(#integer, tPageID)
  tMsg.addProp(#integer, tOfferCode)
  tMsg.addProp(#string, string(tExtraParam))
  tMsg.addProp(#integer, tAsGift)
  if tAsGift then
    tMsg.addProp(#string, tGiftReceiver)
    tMsg.addProp(#string, tGiftMessage)
  end if
  getConnection(getVariable("connection.info.id", #Info)).send("PURCHASE_FROM_CATALOG", tMsg)
end
```

Here we can just read an wire/VL64 encoded integer in our server for #integer and base64-prefixed string for #string. This method is much more easier to use and cleaner as well.

Habbo still doesn't do this for every packet, so every packet has to be cleaned up and using this method.