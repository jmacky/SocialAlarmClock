<html lang="en">

<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.0.0/dist/css/bootstrap.min.css" rel="stylesheet" integrity="sha384-wEmeIV1mKuiNpC+IOBjI7aAzPcEZeedi5yW5f2yOq55WWLwNGmvvx4Um1vskeMj0" crossorigin="anonymous">
    <title>Social Alarm Clock</title>
    <style>
        body { margin-top: 50px; font-family: "Eras ITC"; background-color: #333333; color: #55d400; }
        .time { font-family: "Agency FB"; font-size: 80px; line-height: 1.0; margin-bottom: 45px; }
        .form-switch { margin-bottom: 70px; }
        .form-check-input { background-color: #dadada }
        .form-check-input:checked { background-color: #55d400 }
        #advancedArea label { margin-top: 10px; }
        #updateAlarmTime { margin-left: 60px; margin-top: -35px; visibility: hidden; }
        #offline { position: absolute; top: 25px; right: 50px; color: #d7190c; visibility: hidden; }
        .dot { height: 12px; width: 12px; background-color: #d7190c; border-radius: 50%; display: inline-block; }
    </style>
</head>

<body>

    <!-- Password prompt -->
    <div class="modal" tabindex="-1" id="passwordModal">
        <div class="modal-dialog">
            <div class="modal-content">
                <div class="modal-header">
                    <h5 class="modal-title">Enter password</h5>
                </div>
                <div class="modal-body">
                    <input type="password" name="password" id="password" />&nbsp;&nbsp;
                    <span id="badPassword" style="color: red; visibility: hidden;">not authorized</span>
                </div>
                <div class="modal-footer">
                    <button type="button" class="btn btn-primary" id="submitPassword">ok</button>
                </div>
            </div>
        </div>
    </div>

    <div id="main" class="container">

        <!-- Current time -->
        <div class="row">
            <div class="col"></div>
            <div class="col">
                <div>Current time</div>
                <div class="time" id="currentTime">--:-- am</div>
            </div>
            <div class="col"></div>
        </div>

        <!-- Alarm time -->
        <div class="row align-items-end">
            <div class="col"></div>
            <div class="col">
                <div>Alarm</div>
                <div class="time" contentEditable="true" id="alarmTime">--:-- am</div>
                <div><button class="btn btn-sm btn-primary" id="updateAlarmTime">update</button></div>
            </div>
            <div class="col">
                <div class="form-check form-switch">
                    <input class="form-check-input" type="checkbox" id="alarmEnabled">
                    <label class="form-check-label" for="alarmEnabled" id="alarmEnabledLabel">alarm is off</label>
                </div>
            </div>
        </div>

        <!-- Device offline error -->
        <div id="offline"><span class="dot"></span>&nbsp;&nbsp;Device disconnected</div>

        <!-- Advanced settings -->
        <div class="row">
            <p>
                <button class="btn btn-sm btn-outline-secondary" type="button" data-bs-toggle="collapse" data-bs-target="#advancedArea" aria-expanded="false" aria-controls="advancedArea">
                    Advanced...
                </button>
            </p>
            <div class="collapse container" id="advancedArea">
                <div class="row">
                    <div class="col-6 col-sm-1"><label>gmtOffset:</label></div>
                    <div class="col-6 col-sm-1"><label id="gmtOffset">-</label></div>
                    <!-- force line break --><div class="w-100" style="margin-bottom: 20px;"></div>
                    <div>
                        Command examples
                        <ul>
                            <li>t615. Sets the alarm time to 6:15 am</li>
                            <li>t1930. Sets the alarm time to 7:30 pm</li>
                            <li>a1. Enables the alarm</li>
                            <li>a0. Disables the alarm</li>
                            <li>g-4. Set the time zone to GMT-4</li>
                            <li>b1. Immediately sets off the alarm</li>
                            <li>rFF00FF. Set the RGB LED to purple (i.e. #FF00FF)</li>
                        </ul>
                    </div>
                    <div class="col-6 col-sm-1"><label>cmd:</label></div>
                    <div class="col-6 col-sm-3"><input type="text" class="form-control" id="cmd" /></div>
                    <div class="col-6 col-sm-1"><button class="btn btn-primary" type="button" id="send">send</button></div>
                    <div class="col-6 col-sm-1"><label id="cmdResponse"></label></div>
                    <!-- force line break --><div class="w-100" style="margin-bottom: 20px;"></div>
                    <div><a href="SocialAlarmClockConfig.exe">Run configuration utility</a></div>
                </div>
            </div>
        </div>
    </div>

    <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.0.0/dist/js/bootstrap.bundle.min.js" integrity="sha384-p34f1UUtsS3wqzfto5wAAmdvj+osOnFyQFpp4Ua3gs/ZVWx6oOypYoCJhGGScy+8" crossorigin="anonymous"></script>
    <script src="https://code.jquery.com/jquery-3.6.0.min.js" integrity="sha256-/xUj+3OJU5yExlq6GSYGSHk7tPXikynS7ogEvDej/m4=" crossorigin="anonymous"></script>
    <script src="blynk-browser.min.js"></script>

    <script>
        //-----------------
        //     Globals
        //-----------------
        var id = "FUlbGH5YSCsoAoSsgLbKuejXsOdUVvuB";            //device ID
        var blynk;                                              //Blynk helper
        var editingAlarm = false;                               //Flag that says whether alarm time is in edit more or not
        var devicePassword;                                     //When sending commands to the device

        //-------------------
        //     FUNCTIONS
        //-------------------
        //Parse a time string, like "6:30 am" into a a JSON object with 24-hr hours & minutes.
        //Returns null if failed to parse
        function parseTime(s) {
            //Format must be h:mm ap
            parts = s.split(" ");
            if (parts.length != 2) return null;

            //Get hours & minutes
            time = parts[0];
            digits = time.split(":");
            if (digits.length != 2) return null;
            hour = parseInt(digits[0]);
            minute = parseInt(digits[1]);

            //Get am\pm and set hour accordingly
            ap = parts[1].toLowerCase();
            if (ap == "a" || ap == "am") {
                //fine
            }
            else if (ap == "p" || ap == "pm") {
                if (hour < 12) hour += 12;
            }
            else {
                return null;
            }

            return { "hour": hour, "minute": minute };
        }


        //Convert a time number like 1234 into "12:34 pm"
        function prettyTime(num) {
            hours = parseInt(Math.trunc(num / 100));
            minutes = parseInt(num - (hours * 100));
            ap = "am";

            if (hours > 12) {
                hours -= 12;
                ap = "pm";
            }
            else if (hours == 0) {
                hours = 12;
            }

            if (minutes < 10) {
                minutes = "0" + minutes;
            }

            return hours + ":" + minutes + " " + ap;
        }


        //Get Blynk set up
        function initBlynk() {
            blynk = new Blynk.Blynk(id);
            var currentTime = new blynk.VirtualPin(0);
            var gmtOffset = new blynk.VirtualPin(1);
            var alarmTime = new blynk.VirtualPin(2);
            var alarmEnabled = new blynk.VirtualPin(3);
            //var terminal = new blynk.VirtualPin(11);

            blynk.on('connect', function () {
                console.log("Blynk ready. Sending sync request...");
                blynk.syncAll();
            });

            blynk.on('disconnect', function () {
                console.log("Blynk disconnected.");
            });

            currentTime.on('write', function (param) {
                //console.log("time: " + param[0]);
                $("#currentTime").text(prettyTime(param[0]));
            });

            alarmTime.on('write', function (param) {
                //console.log("alarm: " + param[0]);
                if (!editingAlarm) {
                    $("#alarmTime").text(prettyTime(param[0]));
                }
            });

            alarmEnabled.on('write', function (param) {
                //console.log("alarm set: " + param[0]);
                setAlarmEnabledUI(param[0] == 1);
            });

            gmtOffset.on('write', function (param) {
                //console.log("offset: " + param[0]);
                $("#gmtOffset").text(param[0]);
            });
        }


        //Show in the UI that the alarm is enabled or disabled
        function setAlarmEnabledUI(state) {
            $("#alarmEnabled").prop("checked", state);
            $("#alarmEnabledLabel").text(state ? "alarm is on" : "alarm is off");
        }


        //Send a command to the device via Blynk and wait either a success response (ok-foo),
        //a failure (err-foo), or a timeout, which is treated like an error.
        function sendcmd(cmd, onSuccess, onError, onFinally) {
            var rx = new blynk.VirtualPin(11);

            var timeout = setTimeout(function () {
                rx.removeListener("write", callback);
                if (onError != undefined) onError();
            }, 5000);

            const callback = (param) => {
                var completed = false;

                if (param[0] == "ok-" + cmd) {
                    completed = true;
                    if (onSuccess != undefined) onSuccess();
                }
                else if (param[0] == "err") {
                    completed = true;
                    if (onError != undefined) onError();
                }
                //else not a response we care about

                if (completed) {
                    console.log(param[0]);
                    rx.removeListener("write", callback);
                    clearTimeout(timeout);
                    if (onFinally != undefined) onFinally();
                }
            }

            rx.on("write", callback);

            console.log("Send " + cmd);
            $.get("http://blynk-cloud.com/" + id + "/update/V11?value=" + devicePassword + "+" + cmd);
        }


        //Check hardware connectivity and update UI appropriately
        /*Blynk seems to always say it's online...
        function checkOnline()
        {
            $.get("http://blynk-cloud.com/" + id + "/isHardwareConnected").done(function (response)
            {
                var online = response == "true";
                $("#offline").css("visibility", online ? "hidden" : "visible");
            });
        }
        */

        //-----------------------
        //    EVENT HANDLERS
        //-----------------------
        //Alarm time on focus
        $("#alarmTime").on("focus", function () {
            $("#updateAlarmTime").css("visibility", "visible");
            editingAlarm = true;
        });


        //Update alarm time button
        $("#updateAlarmTime").click(function () {
            t = parseTime($("#alarmTime").text());
            if (t != null) {
                var time = t.hour * 100 + t.minute;
                var cmd = "t" + time;
                $("#updateAlarmTime").css("visibility", "hidden");
                $("#alarmTime").text("sending...");
                sendcmd(cmd,
                    /*on success*/ function () { $("#alarmTime").text(prettyTime(time)); },
                    /*on error*/   function () { alert("error"); },
                    /*finally*/    function () { editingAlarm = false; }
                );
            }
            else {
                alert("invalid time");
            }
        });


        //Alarm enabled switch
        $("#alarmEnabled").on("change", function () {
            var state = $("#alarmEnabled").prop("checked");
            var cmd = state ? "a1" : "a0";
            editingAlarm = true;
            $("#alarmEnabledLabel").text("sending...");
            sendcmd(cmd,
                /*on success*/ function () { setAlarmEnabledUI(state) },
                /*on error*/   function () { alert("error"); },
                /*finally*/    function () { editingAlarm = false; }
            );
        });


        //Send cmd button
        $("#send").click(function () {
            $("#cmdResponse").text("");
            $("#send").text("sending...");
            sendcmd($("#cmd").val(),
                /*on success*/ function () { $("#cmdResponse").text("ok"); },
                /*on error*/   function () { $("#cmdResponse").text("err"); },
                /*finally*/    function () { $("#send").text("send"); }
            );
            $("#cmd").val("");
            blynk.syncAll();
        });



        //-----------------
        //     ON LOAD
        //-----------------
        //Start Blynk and kick off timers to periodically read values.
        initBlynk();
        setInterval(function () { blynk.syncAll(); }, 3000);

        //Prompt for password
        var modalElement = document.getElementById("passwordModal");
        var passwordModal = new bootstrap.Modal(modalElement);
        passwordModal.show();
        $("#submitPassword").click(function (event)
        {
            $("#submitPassword").text("sending...");
            devicePassword = $("#password").val();
            sendcmd("n0", //no-op
                /*on success*/ function () { passwordModal.hide(); },
                /*on error*/   function () { $("#badPassword").css("visibility", "visible"); },
                /*finally*/    function () { $("#submitPassword").text("ok"); }
            );
        });
    </script>
</body>
</html>
