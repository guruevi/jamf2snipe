# jamf2snipe

## Import/Sync Computers from JAMF to Snipe-IT

```
usage: jamf2snipe [-h] [-v] [--auto_incrementing] [--dryrun] [-d] [--do_not_update_jamf] [--do_not_verify_ssl] [-f] [--version] [-u | -uf] [-m | -c]

options:
  -h, --help            show this help message and exit
  -v, --verbose         Sets the logging level to INFO and gives you a better idea of what the script is doing.
  --auto_incrementing   You can use this if you have auto-incrementing enabled in your snipe instance to utilize that instead of adding the Jamf ID for the asset tag.
  --dryrun              This checks your CONFIG and tries to contact both the JAMFPro and Snipe-it instances, but exits before updating or syncing any assets.
  -d, --debug           Sets logging to include additional DEBUG messages.
  --do_not_update_jamf  Do not update Jamf with the asset tags from Snipe.
  --do_not_verify_ssl   Skips SSL verification for all requests. Helpful when you use self-signed certificate.
  -f, --force           Updates the Snipe asset with information from Jamf every time, despite what the timestamps indicate.
  --version             Prints the version and exits.
  -u, --users           Checks out the item to the current user in Jamf if it's not already deployed
  -uf, --users_force    Checks out the item to the current user in Jamf even if it's already deployed
  -m, --mobiles         Runs against the Jamf mobiles endpoint only.
  -c, --computers       Runs against the Jamf computers endpoint only.
```

## Overview:

This tool will sync assets between a JAMF Pro instance and a Snipe-IT instance. The tool searches for assets based on
the serial number, not the existing asset tag. If assets exist in JAMF and are not in Snipe-IT, the tool will create an
asset and try to match it with an existing Snipe model. This match is based on the Mac's model identifier (ex.
MacBookAir7,1) being entered as the model number in Snipe, rather than the model name. If a matching model isn't found,
it will create one.

When an asset is first created, it will fill out only the most basic information. When the asset already exists in your
Snipe inventory, the tool will sync the information you specify in the settings.conf file and make sure that the
asset_tag field in JAMF matches the asset tag in Snipe, where Snipe's info is considered the authority.

> Because it determines whether JAMF or Snipe has the most recently updated record, there is the potential to have bad
> data from Jamf overwrite good data in Snipe (ex. purchase date). You can override this with -f or --force to force.

Lastly, if the asset_tag field is blank in JAMF when it is being created in Snipe, it will use JAMFID-<jamfid#> as the
asset tag in Snipe. This way, you can easily filter this out and run scripts against it to correct in the future.

## Requirements:

- Python3 (3.9 or higher) is installed on your system (preferrably in a virtual environment)
- Network access to both your JAMF and Snipe-IT environments.
- A JAMF username and password that has read & write permissions for computer assets and mobile device assets
- Snipe API key for a user that has edit/create permissions for assets and models. Snipe-IT documentation instructions
  for creating API
  keys: [https://snipe-it.readme.io/reference#generating-api-tokens](https://snipe-it.readme.io/reference#generating-api-tokens)

## Installation:

### Mac

1. Install Python 3.9 or later

- Grab the latest PKG installer from the Python website and run it.
- Use Homebrew to install Python 3.9 or later: `brew install python3`

2. Add Python to your PATH

- Run the `Update Shell Profile.command` script in the `/Applications/Python 3.X` folder to add `python3.X` your PATH

3. Create a virtualenv for jamf2snipe

- Create the virtualenv: `mkdir ~/.virtualenv`
- Add `python3.X` to the virtualenv: `python3.X -m venv ~/.virtualenv/jamf2snipe`
- Activate the virtualenv: `source ~/.virtualenv/jamf2snipe/bin/activate`

4. Install dependencies

- `pip install -r /path/to/jamf2snipe/requirements.txt`

5. Configure settings.conf (you can start by copying settings.conf.example to settings.conf)
6. Run `python jamf2snipe` & profit

### Linux

1. Copy the files to your system (recommend installing to /opt/jamf2snipe/* ). Make sure you meet all the system
   requirements.
2. Edit the settings.conf to match your current environment - you can start by copying settings.conf.example to
   settings.conf. The script will look for a valid settings.conf in /opt/jamf2snipe/settings.conf,
   /etc/jamf2snipe/settings.conf, or in the current folder (in that order): so either copy the file to one of those
   locations, or be sure that the user running the program is in the same folder as the settings.conf.

## Configuration - settings.conf:

See the [settings.conf.example](https://github.com/grokability/jamf2snipe/blob/main/settings.conf.example) for required
keys. You will need valid subsets from [JAMF's API](https://developer.jamf.com/apis/classic-api/index) to associate
fields into Snipe.

### Required

Note: do not add `""` or `''` around any values.

**[jamf]**

- `url`: https://*your_jamf_instance*.com:*port*
- `username`: Jamf API user username
- `password`: Jamf API user password

**[snipe-it]**

Check out the [settings.conf.example](https://github.com/grokability/jamf2snipe/blob/main/settings.conf.example) file
for the full documentation

- `url`: http://*your_snipe_instance*.com
- `apikey`: API key generated via [these steps](https://snipe-it.readme.io/reference#generating-api-tokens).
- `manufacturer_id`: The manufacturer database field id for the Apple in your Snipe-IT instance. You will probably have
  to create a Manufacturer in Snipe-IT and note its ID.
- `default_status_id`: The status database field id to assign to any assets created in Snipe-IT from JAMF. Usually you
  will want to pick a status like "Ready To Deploy" (2) - look up its ID in Snipe-IT and put the ID here.
- `computer_model_category_id`: The ID of the category you want to assign to JAMF computers. You will have to create
  this in Snipe-IT and note the Category ID
- `mobile_model_category_id`: The ID of the category you want to assign to JAMF mobile devices. You will have to create
  this in Snipe-IT and note the Category ID

### API Mapping

To get the database fields for Snipe-IT Custom Fields, go to Custom Fields, scroll down past Fieldsets to Custom Fields,
click the column selection and button and select the unchecked 'DB Field' checkbox. Copy and paste the DB Field name for
the Snipe under api-mapping in settings.conf.

To get the database fields for Jamf, refer to
Jamf's ["Jamf Pro" API documentation](https://developer.jamf.com/jamf-pro/reference/jamf-pro-api).

You need to set the manufacturer_id for Apple devices in the settings.conf file. To get this, go to Manufacturers, click
the column selection button and select the `ID` checkbox.

Some example API mappings for computers can be
found in [this call](https://developer.jamf.com/jamf-pro/reference/get_v1-computers-inventory).

```
name = general name
_snipeit_mac_address_1 = hardware macAddress
_snipeit_ram_2 = hardware totalRamMegabytes
_snipeit_storage_3 = storage disks 0 sizeMegabytes
_snipeit_mac_address_2_5 = hardware altMacAddress
_snipeit_operating_system_8 = operatingSystem name
_snipeit_os_version_9 = operatingSystem version
_snipeit_os_build_10 = operatingSystem build
_snipeit_ip_address_13 = general lastIpAddress
_snipeit_cpu_name_14 = hardware processorType
```

If you have data such as purchase date in JAMF, you can map it as such:

```
purchase_date = purchasing purchased 
warrantY_date = purchasing warrantyDate 
```

Some example API mappings for mobile devices can be
found in [this call](https://developer.jamf.com/jamf-pro/reference/get_v2-mobile-devices-detail).

```
_snipeit_imei_4 = network imei
_snipeit_mac_address_1 = hardware wifiMacAddress
_snipeit_storage_3 = hardware capacityMb
_snipeit_mac_address_2_5 = hardware bluetoothMacAddress
_snipeit_operating_system_8 = deviceType
_snipeit_os_version_9 = general osVersion
_snipeit_os_build_10 = general osBuild
_snipeit_ip_address_13 = general ipAddress
name = general displayName
```

## Testing

It is *always* a good idea to create a test environment to ensure everything works as expected before running anything
in production.

Because `jamf2snipe` only ever writes the asset_tag for a matching serial number back to Jamf, testing with your
production JAMF Pro is OK. However, this can overwrite good data in Snipe. You can spin up a Snipe instance in Docker
pretty quickly ([see the Snipe docs](https://snipe-it.readme.io/docs/docker)).

## Contributing

Thanks to all the people that have already contributed to this project! If you have something you'd like to add
please help by forking this project then creating a pull request to the `devel` branch. When working on new features,
please try to keep existing configs running in the same manner with no changes. When possible, open up an issue and
reference it when you make your pull request.
