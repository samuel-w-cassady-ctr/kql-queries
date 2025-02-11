let ProductNameMapping = externaldata(VID: string, Vendor: string, PID: string, ProductName: string)
    [h@"https://raw.githubusercontent.com/samuel-w-cassady-ctr/usb-hunt/refs/heads/main/VID-PID-List.txt"]
    with (format="csv", ignoreFirstRecord=true)
    | extend VID = tostring(VID)
    | extend Vendor = tostring(Vendor)
    | extend PID = tostring(PID)
    | extend ProductName = tostring(ProductName)
    | project VID, Vendor, PID, ProductName;
let loggedOnUser = DeviceInfo
    | extend data=parsejson(LoggedOnUsers)
    | mv-expand data
    | extend userName = data.UserName
    | project userName, DeviceId, Timestamp;
DeviceEvents
    | where DeviceName startswith "CARS"
    | where ActionType == 'PnpDeviceConnected'
    | extend PNP = parsejson(AdditionalFields)
    | extend PnPDeviceId = PNP.DeviceId
    | extend DeviceDescription = PNP.DeviceDescription
    | where PnPDeviceId startswith @'USB\'
    | where not(DeviceDescription has_any("Hub", "Dock", "Controller", "Realtek", "Headset",
        "Audio", "Camera", "Cam", "Smart", "Microsoft", "Printer", "HP", "Vendor", "Adesso", "CyberTrack", "Brother", "Zebra", "Docking Station", "WinUSB", "laserjet", "Reader", "Lexmark", "billboard", "Adapter",
        "HID", "AV", "USB2.0", "Canon", "Smartboard", "Webcam", "Dell", "USBAudio", "ADB", "DeskJet", "Extron", "OMNIKEY", "Video"))
    | where not(PnPDeviceId has_any("VID_046D", "VID_413C", "VID_0000", "VID_03F0", "VID_043D", "VID_045E", "VID_0424", "VID_0BDA", "VID_0B0E", "VID_093A")) 
    | extend Device_VID = substring(tolower(PnPDeviceId), indexof(tolower(PnPDeviceId), "vid_") + 4, 4)
    | extend Device_PID = substring(tolower(PnPDeviceId), indexof(tolower(PnPDeviceId), "pid_") + 4, 4)
    | join kind=leftouter (ProductNameMapping) on $left.Device_PID == $right.PID and $left.Device_VID == $right.VID
    | project
        Timestamp,
        DeviceName,
        ActionType,
        DeviceId,
        Device_VID,
        Device_PID,
        DeviceDescription,
        PnPDeviceId,
        Vendor,
        ProductName,
        ReportId
    | join kind=inner (loggedOnUser) on DeviceId
    | summarize arg_max(DeviceName, *) by DeviceName
    | project-away DeviceId1, Timestamp1  
    | sort by Timestamp desc
