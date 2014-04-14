BeaconScanner
=============

iBeacon Scanning Utility for OSX

![Alt text](ScreenShot.png)

A notible absence when apple added iBeacon support to iOS was a lack of a client API and utilities for the Mac desktop.  This application attempts to remedy this, allowing an easy way to scan for beacons from a desktop as well as providing an underlying the source framework for adding iBeacon client support to any other OSX project.


##How to install a prebuilt binary.

To install without building from source, first [download the prebuilt archive](Builds/BeaconScanner.zip).  Double click the zip to extract, and then double click again to run. 

Once you start the app it'll automatically begin scanning for bluetooth devices.  Any beacons within range will automatically appear and it will continuously update as long as it remains scanning.

##How to build from source.

Building the app requires [cocoapods](http://cocoapods.org).  Once installed, launch Terminal.app and in the project directory, run "pod install".  When it completes, open the *BeaconScanner.xcworkspace* file that it create in Xcode.  The app should then build and run successfully. 


##How to add beacon support to your own project

To add iBeacon support to your own desktop application (at least until the a proper cocoapod is made available), just copy the following four files into your project:  

- *HGBeaconManager.h*
- *HGBeaconManager.m*
- *HGBeacon.h*
- *HGBeacon.m*

The beacon manager announces the beacons it detects to the subscribers of the [ReactiveCocoa](https://github.com/ReactiveCocoa/ReactiveCocoa) signal that it provides for this.  

All that's needed for a client to detect beacons is to subscribe to this signal:

	[[[HGBeaconManager] sharedBeaconManager] startScanning];
	
	RACSignal *beaconSignal = [[HGBeaconManager sharedBeaconManager] beaconSignal];
	[beaconSignal subscribeNext:^(HGBeacon *detectedBeacon) {
		NSLog(@"iBeacon Detected: %@", detectedBeacon);
	}];


To limit this subscription to just those beacons that are relevant to your application, a filtered signal can be composed from the raw feed, like so:


	NSUUID *applicationUUID = [[NSUUID alloc] initWithUUIDString:@"B9407F30-F5F8-466E-AFF9-25556B57FE6D"]

	RACSignal *filteredSignal = [[[HGBeaconManager sharedBeaconManager] beaconSignal] filter:^(HGBeacon *beacon) {
		return [beaconSignal.proximityUUID isEqual:applicationUUID];
	}];
	

Note that as long as any given beacon is in range, it will be announced periodically via the subscription.  

It's best to use these updates to maintain a seperate list of nearby active beacons, and then periodically purge from this list those that have not been heard from in a while in order to ensure that only active beacons are tracked.   

In the application source, the class *HGBeaconViewController* provides a good example of how to do this. 

##How it works

The iBeacon protocol is actually pretty simple.  It embeds a formatted payload into the manufacturer data field of a bluetooth LE advertisment. 

In order to receive these advertisements *HGBeaconManager* instantiance an instance of a Core Bluetooth Central manager, and assigns it a dedicated queue for handling events:

	self.managerQueue = dispatch_queue_create("com.huge.DesktopBeacon.centralManagerQueue", NULL);
    self.centralManager = [[CBCentralManager alloc] initWithDelegate:self
                                                               queue:self.managerQueue];

The application begins listening for all such advertisements by asking the Bluetooth Central Manager to start scanning for peripherals:

	[self.centralManager scanForPeripheralsWithServices:nil
                                                options:@{ CBCentralManagerScanOptionAllowDuplicatesKey : @YES}];


The options specify it should invoke its scan callback when the same device detected on a scan on multiple occassions.  This is important for iBeacons, as their continued detection is the only way to determine if they're still in range. 

In the callback, the beacon manager determines whether or not a detected peripheral is an iBeacon by seeing if it can be parsed as such from the advertisement data dictionary recieved.  If so, it sends the detected beacon to its subscribers:


	- (void)centralManager:(CBCentralManager *)central
	 didDiscoverPeripheral:(CBPeripheral *)peripheral
 	    advertisementData:(NSDictionary *)advertisementData
    	              RSSI:(NSNumber *)RSSI {
    	HGBeacon *beacon = [HGBeacon beaconWithAdvertismentDataDictionary:advertisementData];
	    beacon.RSSI = RSSI;
	    if (beacon) {
    	    [(RACSubject *)self.beaconSignal sendNext:[beacon copy]];
	    }
	}

The parsing of the advertisement data dictionary to determine if a peripheral is an iBeacon happens inside *HGBeacon*.  It first pulls the manufacturer data from the advertisement dictionary and then checks and parses this payload.  If it succeeds, it returns a beacon, if it fails, a *nil*. 



	+(HGBeacon *)beaconWithAdvertismentDataDictionary:(NSDictionary *)advertisementDataDictionary {
	    NSData *data = (NSData *)[advertisementDataDictionary objectForKey:CBAdvertisementDataManufacturerDataKey];
	    if (data) {
	        return [self beaconWithManufacturerAdvertisementData:data];
	    }
	    return nil;
	}

	+(HGBeacon *)beaconWithManufacturerAdvertisementData:(NSData *)data {
	    if ([data length] != 25) {
	        return nil;
	    }

	    u_int16_t companyIdentifier,major,minor = 0;
	    int8_t measuredPower,dataType, dataLength = 0;
	    char uuidBytes[17] = {0};
	    NSRange companyIDRange = NSMakeRange(0,2);
	    [data getBytes:&companyIdentifier range:companyIDRange];
	    if (companyIdentifier != 76) {
	        return nil;
	    }
	    NSRange dataTypeRange = NSMakeRange(2,1);
	    [data getBytes:&dataType range:dataTypeRange];
	    if (dataType != 0x02) {
	        return nil;
	    }
	    NSRange dataLengthRange = NSMakeRange(3,1);
	    [data getBytes:&dataLength range:dataLengthRange];
	    if (dataLength != 0x15) {
	        return nil;
	    }
	    NSRange uuidRange = NSMakeRange(4, 16);
	    NSRange majorRange = NSMakeRange(20, 2);
	    NSRange minorRange = NSMakeRange(22, 2);
	    NSRange powerRange = NSMakeRange(24, 1);
    	[data getBytes:&uuidBytes range:uuidRange];
	    NSUUID *proximityUUID = [[NSUUID alloc] initWithUUIDBytes:(const unsigned char*)&uuidBytes];
	    [data getBytes:&major range:majorRange];
	    major = (major >> 8) | (major << 8);
	    [data getBytes:&minor range:minorRange];
	    minor = (minor >> 8) | (minor << 8);
	    [data getBytes:&measuredPower range:powerRange];
	    HGBeacon *beaconAdvertisementData = [[HGBeacon alloc] initWithProximityUUID:proximityUUID
	                                                                          major:[NSNumber numberWithUnsignedInteger:major]
	                                                                          minor:[NSNumber numberWithUnsignedInteger:minor]
	                                                                  measuredPower:[NSNumber numberWithShort:measuredPower]];
	    return beaconAdvertisementData;
	}



That's really it.   For more detailed information on the contents of the payload, see the articles below.

## See Also

An excellent primer is [Adam Warski's post on "How iBeacons work?"](http://www.warski.org/blog/2014/01/how-ibeacons-work/).

The ["What is the iBeacon Bluetooth Profile"](http://stackoverflow.com/questions/18906988/what-is-the-ibeacon-bluetooth-profile) thread on Stack Overflow is also very informative.

In order to turn your OSX Mavericks box into an iBeacon emitter, see Matthew Robinsons' [BeaconOX](https://github.com/mttrb/BeaconOSX). It's also the reason this project exists. 



### Addendum

The "Radar" icon image the utility uses was created by [ricaodomaia](http://openclipart.org/user-detail/ricardomaia) and downloaded from [openclipart.org](http://openclipart.org/detail/122719/radar-by-ricardomaia) 