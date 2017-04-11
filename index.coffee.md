This is a complete, runnable example for `useful-wind` with a Docker.io FreeSwitch image.
The Docker image will run on your local interface (Docker's `--net host`) so you should be able to test with a standard SIP client.

Usage
-----

Assuming you have recent Node.js and Docker.io installed:

```shell
./run.sh
```

Application
-----------

The script doesn't do much besides issuing `answer` and then `sleep` commands, and logging DTMF.

    Server = require 'useful-wind/call_server'
    isset = require 'isset'
    dateFormat = require 'dateformat'
    urlSocket ='http://localhost:8010'
    ivrBase ='/usr/local/freeswitch/sounds/en/us/callie'
    arrEvents =['PinAccepted','InvalidPin','AdminNotCome','ParticipantAdded']
    arrHangupEvent =['NoConfDate', 'BalanceFinished', 'NoCreditInAccount', 'AccountBlocked', 'ConfLocked']
    eventHandler =''
    sessionObj =''
    io = require 'socket.io-client'
    socket = io.connect urlSocket, {reconnect: true, forceNew: true}
    socket.on 'connect', () ->
       console.log "Connected! with server port * 8010"
    socket.on 'disconnect', ()->
       console.log 'Server is disconnected!'
       socket.close()

    test =
      include: (ctx) ->
        @call.on 'DTMF', (res) ->
          digit = res.body['DTMF-Digit']
          console.log "Received #{digit}"
          if ctx.session[this.data['Unique-ID']].IsJoinConf is 0
             ctx.session[this.data['Unique-ID']].DTMF.push(digit)
             if digit is '#'
                arrDTMF = ctx.session[this.data['Unique-ID']].DTMF
                index = arrDTMF.indexOf("#")
                if index > -1
                     arrDTMF.splice(index, 1)
                final_pin = arrDTMF.join("")
                console.log "Final PIN === #{final_pin}"
                ctx.session[this.data['Unique-ID']].DTMF =[]
                jsonObj = {
                    CallerUsername : this.data['Caller-Username'],
                    DID : this.data['Caller-Destination-Number'],
                    reqType: "validatePIN",
                    PIN: final_pin,
                    UniqueID: this.data['Unique-ID'],
                    module: "AC"
                  }
                socket.emit 'validatePIN', jsonObj
                console.log "Sent request for PIN Validate"
          else
             console.log "You have already in conference"

        @action 'answer'
        .then ->
        eventHandler = @action
        sessionObj = ctx.session
        sessionObj[this.data['Unique-ID']] ={
          PinAccepted:{appUUID:'', appData:"{#{this.data['Unique-ID']},PinAccepted}"},
          InvalidPin:{appUUID:'', appData:"{#{this.data['Unique-ID']},InvalidPin}"},
          NoConfDate:{appUUID:'', appData:"{#{this.data['Unique-ID']},NoConfDate}"},
          BalanceFinished:{appUUID:'', appData:"{#{this.data['Unique-ID']},BalanceFinished}"},
          NoCreditInAccount:{appUUID:'', appData:"{#{this.data['Unique-ID']},NoCreditInAccount}"},
          AccountBlocked:{appUUID:'', appData:"{#{this.data['Unique-ID']},AccountBlocked}"},
          ConfLocked:{appUUID:'', appData:"{#{this.data['Unique-ID']},ConfLocked}"},
          AdminNotCome:{appUUID:'', appData:"{#{this.data['Unique-ID']},AdminNotCome}"},
          ConfrenceStablished:{appUUID:'', appData:"{#{this.data['Unique-ID']},ConfrenceStablished}"},
          ParticipantAdded:{appUUID:'', appData:"{#{this.data['Unique-ID']},ParticipantAdded}"},
          DTMF:[],
          arrDTMFResponse:{},
          UniqueID:this.data['Unique-ID'],
          IsJoinConf:0
        }
        @action 'playback', "#{ivrBase}/ivr/8000/ivr-welcome_to_freeswitch.wav"
        .then ->
            console.log "Welcome ivr complete."
            jsonObj = {
              callerUser : this.data['Caller-Username'],
              DID : this.data['Caller-Destination-Number'],
              reqType: "validateDID",
              UniqueID: this.data['Unique-ID'],
              module: "AC"
            }
            socket.emit 'validateDID', jsonObj
            console.log "Send request to DID validate."

        @call.on 'CHANNEL_EXECUTE', (res) ->
            respBody =res.body
            appData = if isset(respBody['Application-Data']) then respBody['Application-Data'] else ''
            appUUID =respBody['Application-UUID']
            arrSession = ctx.session
            arrUserInfo = arrSession[this.data['Unique-ID']]
            if appData
              arrAppData =appData.split('/')
              if isset(arrAppData[0]) && arrAppData[0]
                 appDataPart = arrAppData[0].replace( /[{}]/g, '' )
                 arrAppDataCheck =appDataPart.split(',')
                 if arrAppDataCheck[0] && arrAppDataCheck[1]
                    if arrAppDataCheck[0] is this.data['Unique-ID'] && arrAppDataCheck[1] in arrEvents
                       arrUserInfo[arrAppDataCheck[1]].appUUID = appUUID

        @call.on 'CHANNEL_EXECUTE_COMPLETE', (res) ->
           respBody = res.body
           appData = if isset(respBody['Application-Data']) then respBody['Application-Data'] else ''
           appUUID =respBody['Application-UUID']
           arrSession = ctx.session
           arrUserInfo = arrSession[this.data['Unique-ID']]
           if appData
             arrAppData =appData.split('/')
             if isset(arrAppData[0]) && arrAppData[0]
                appDataPart = arrAppData[0].replace( /[{}]/g, '' )
                arrAppDataCheck =appDataPart.split(',')
                if arrAppDataCheck[0] && arrAppDataCheck[1] && arrAppDataCheck[1] in arrEvents
                  if arrAppDataCheck[0] is this.data['Unique-ID']  && arrUserInfo[arrAppDataCheck[1]].appUUID
                     arrUserInfo[arrAppDataCheck[1]].appUUID = ''
                     if arrAppDataCheck[1] is 'PinAccepted'
                        console.log "PIN hase been accepted successfully"
                        arrDTMFResponse = arrUserInfo.arrDTMFResponse
                        confFlowStatus =1
                        now = new Date()
                        curentDate = dateFormat(now, "yyyy-mm-dd HH:MM:ss")
                        if isset(arrDTMFResponse.ac_conf.starttime) && isset(arrDTMFResponse.ac_conf.endtime)
                           console.log "Starttime and endtime is set"
                           if arrDTMFResponse.ac_conf.endtime >= curentDate && arrDTMFResponse.ac_conf.starttime <= curentDate
                               console.log "Date and Time is okay for conf of Reserver one"
                           else
                                console.log "Date and Time is NOT okay for conf of Reserver one"
                                confFlowStatus =0
                                server.fs_command "uuid_broadcast #{this.data['Unique-ID']} {#{this.data['Unique-ID']},NoConfDate}#{ivrBase}/NoConfDate.wav aleg"
                        else
                            console.log "Unreserve Conference so No Date and Time needs"
                        if confFlowStatus is 1
                           if arrDTMFResponse.cc_card.status is 1
                              if arrDTMFResponse.cc_card.typepaid is 1
                                 bal = arrDTMFResponse.cc_card.creditlimit + arrDTMFResponse.cc_card.credit
                                 console.log "Balance= #{bal}"
                                 if bal < 0
                                   console.log "No Balance in your account"
                                   confFlowStatus =0
                                   server.fs_command "uuid_broadcast #{this.data['Unique-ID']} {#{this.data['Unique-ID']},BalanceFinished}#{ivrBase}/balanceFinished.wav aleg"
                              else
                                if arrDTMFResponse.cc_card.credit<=0
                                    console.log "No credit amount in your account"
                                    confFlowStatus =0
                                    server.fs_command "uuid_broadcast #{this.data['Unique-ID']} {#{this.data['Unique-ID']},NoCreditInAccount}#{ivrBase}/balanceFinished.wav aleg"
                           else
                              confFlowStatus =0
                              server.fs_command "uuid_broadcast #{this.data['Unique-ID']} {#{this.data['Unique-ID']},AccountBlocked}#{ivrBase}/AccountBlock.wav aleg"
                           if confFlowStatus is 1
                              if arrDTMFResponse.ac_conf.locked is 1
                                  console.log "conference is locked"
                                  confFlowStatus =0
                                  server.fs_command "uuid_broadcast #{this.data['Unique-ID']} {#{this.data['Unique-ID']},ConfLocked}#{ivrBase}/conference/8000/conf-locked.wav aleg"
                              else
                                 if arrDTMFResponse.ac_conf.adminpin is arrDTMFResponse.PIN
                                   confmanager =1
                                 else
                                   confmanager =0
                                 arrParticipant ={
                                      reqType:'addParticipant',
                                      module: 'AC',
                                      UniqueID: arrDTMFResponse.UniqueID,
                                      arrData:{
                                          uniqueid: arrDTMFResponse.UniqueID,
                                          starttime:dateFormat(now, "yyyy-mm-dd HH:MM:ss"),
                                          endtime:dateFormat(now, "yyyy-mm-dd HH:MM:ss"),
                                          duration:0,
                                          callstatus:1,
                                          callerid:arrDTMFResponse.CallerUsername,
                                          calleridname:arrDTMFResponse.CallerUsername,
                                          callerchannel:arrDTMFResponse.UniqueID,
                                          calleruniqueid:arrDTMFResponse.UniqueID,
                                          id_ac_conf:arrDTMFResponse.ac_conf.id,
                                          confmanager:confmanager,
                                          callername:arrDTMFResponse.CallerUsername,
                                          file:'',
                                          initial_user:0
                                      }
                                  }
                                 socket.emit 'addParticipant', arrParticipant
                     else if arrAppDataCheck[1] is 'InvalidPin'
                        console.log "Invalid PIN, try again!"
                        server.fs_command "uuid_broadcast #{this.data['Unique-ID']} #{ivrBase}/conference/8000/conf-pin.wav aleg"
                  else if arrAppDataCheck[1] in arrHangupEvent
                     if arrAppDataCheck[0] is this.data['Unique-ID']  && arrUserInfo[arrAppDataCheck[1]].appUUID
                         console.log "Phone is hangup"
                         eventHandler 'hangup'
                  else
                     console.log "No Task assign."

        @call.on 'CHANNEL_HANGUP', (res) ->
           channel_data = res.body
           console.log "Call has been cancelled  Unique-ID ==== #{channel_data['Caller-Unique-ID']}"
           sessionObj = ctx.session
           delete sessionObj[channel_data['Caller-Unique-ID']]
           now = new Date()
           curentDate = dateFormat(now, "yyyy-mm-dd HH:MM:ss")
           jsonObj ={
               reqType:'updateParticipant',
               module: 'AC',
               UniqueID: channel_data['Unique-ID'],
               condition:{
                   uniqueid: channel_data['Unique-ID']
                 }
              setValue:{
                  callstatus:0,
                  endtime:curentDate,
                  initial_user:0
              }
            }
           socket.emit 'updateParticipant', jsonObj

        @call.on 'CHANNEL_HANGUP_COMPLETE', (res) ->
           channel_data = res.body
           console.log "Call has been complete  Caller-Unique-ID ==== #{channel_data['Caller-Unique-ID']}"
           sessionObj = ctx.session
           delete sessionObj[channel_data['Caller-Unique-ID']]
           now = new Date()
           curentDate = dateFormat(now, "yyyy-mm-dd HH:MM:ss")
           jsonObj ={
               reqType:'updateParticipant',
               module: 'AC',
               UniqueID: channel_data['Unique-ID'],
               condition:{
                   uniqueid: channel_data['Unique-ID']
                 }
              setValue:{
                  callstatus:0,
                  endtime:curentDate,
                  initial_user:0
              }
            }
           socket.emit 'updateParticipant', jsonObj

Configuration. In this case this only lists the middleware(s) we'll be using.

    cfg =
      use: [
        test
      ]

    server = new Server cfg
    server.listen 7010
    console.log "Middleware is runing on port * 7010"

    socket.on 'validateDID', (response) ->
      console.log response.msg
      if sessionObj[response.UniqueID] && response.UniqueID is sessionObj[response.UniqueID].UniqueID
         if response.status is 'OK'
            server.fs_command "uuid_broadcast #{response.UniqueID} #{ivrBase}/conference/8000/conf-pin.wav aleg"
         else
           eventHandler "playback", "#{ivrBase}/DIDdeactivate.wav"
           .then ->
              console.log "DID ivr complete"
              eventHandler 'hangup'

    socket.on 'validatePIN', (response) ->
         if sessionObj[response.UniqueID] && response.UniqueID is sessionObj[response.UniqueID].UniqueID
           if response.status is 'OK'
               sessionObj[response.UniqueID].arrDTMFResponse =response
               server.fs_command "uuid_broadcast #{response.UniqueID} {#{response.UniqueID},PinAccepted}#{ivrBase}/PinAcceptJoinConfNow.wav aleg"
           else
               sessionObj[response.UniqueID].arrDTMFResponse = {}
               server.fs_command "uuid_broadcast #{response.UniqueID} {#{response.UniqueID},InvalidPin}#{ivrBase}/conference/8000/conf-bad-pin.wav aleg"

    socket.on 'addParticipant', (response) ->
       if sessionObj[response.UniqueID] && response.UniqueID is sessionObj[response.UniqueID].UniqueID
         if response.status is 'OK'
            arrUserInfo = sessionObj[response.UniqueID]
            arrDTMFResponse = arrUserInfo.arrDTMFResponse
            if arrDTMFResponse.ac_conf.recording is 1
                now = new Date()
                curentDate = dateFormat(now, "yyyy_mm_dd")
                RecFileName= "#{arrDTMFResponse.ac_conf.id}_#{arrDTMFResponse.ac_conf.id_cc_card}_#{curentDate}_#{response.initial_user}"
                path="/usr/local/freeswitch/recordings/#{RecFileName}.wav"
                eventHandler "set","conference_auto_record=#{path}"
                # .then ->
                #     console.log "Recording start"
                #     if response.new_user is 1
                #         recordingJsonObj = {
                #           reqType: "recordingSaved",
                #           UniqueID: response.UniqueID,
                #           module: "AC",
                #           arrData:{
                #              location: path,
                #              type: 0,
                #              start_time: dateFormat(now, "yyyy-mm-dd HH:MM:ss"),
                #              id_ac_conf: arrDTMFResponse.ac_conf.id,
                #              server_ip: '192.168.1.230',
                #              duration: 0
                #              }
                #            }
            if arrDTMFResponse.PIN is arrDTMFResponse.ac_conf.userpin
                 feature ='unmute,'
                 if arrDTMFResponse.ac_conf.moh is 1
                     feature ="nomoh,"
                 sessionObj[response.UniqueID].IsJoinConf =1
                 eventHandler "conference","#{arrDTMFResponse.ac_conf.id}\@default++flags{#{feature}}"
                 console.log "Conference started by Participant"
            else if arrDTMFResponse.ac_conf.adminpin is arrDTMFResponse.PIN
                 feature ="unmute,moderator"
                 sessionObj[response.UniqueID].IsJoinConf =1
                 eventHandler "conference","#{arrDTMFResponse.ac_conf.id}\@default++flags{#{feature}}"
                 console.log "Conference started by Admin"
