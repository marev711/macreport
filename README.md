<!-- Reference style links for readability
*** https://www.markdownguide.org/basic-syntax/#reference-style-links
-->
[![Issues][issues-shield]][issues-url]
[![MIT License][license-shield]][license-url]

<br />
<p align="center">
  <a href="https://github.com/marev711/macreport">
    <img src="images/macreport-logo.png" alt="Logo" width="93" height="80">
  </a>

  <h3 align="center">macreport</h3>

  <p align="center">
    Collection of script to run on a Raspberry Pi to monitor and visualize MAC addresses connected to your local WiFi network.
  </p>
</p>



<!-- TABLE OF CONTENTS -->
<details open="open">
  <summary><h2 style="display: inline-block">Table of Contents</h2></summary>
  <ol>
    <li><a href="#about-the-project">About The Project</a></li>
    <li><a href="#getting-started">Getting Started</a></li>
    <li><a href="#prerequisites">Prerequisites</a></li>
    <li><a href="#configuration">Configuration</a></li>
    <li><a href="#use-cases">Use Cases</a></li>
    <li><a href="#roadmap">Roadmap</a></li>
    <li><a href="#contributing">Contributing</a></li>
    <li><a href="#license">License</a></li>
    <li><a href="#contact">Contact</a></li>
    <li><a href="#acknowledgements">Acknowledgements</a></li>
  </ol>
</details>


## About The Project

[![Product Name Screen Shot][product-screenshot]](https://github.com/marev711/macreport/images/macreport-example-image.png)
Macreport is a collection of scripts intended to run on a Raspberry Pi connected to your local WiFi. The project is part of the final project assignment in the Harvard CS50 course, hence only the required README file is provided in this repository. The scripts will record and timestamp MAC-addresses connected to you local WiFi in an SQLITE database, and visualize the recorded entries via a Flask server run on the Pi. Macreport also provides integration to [Pushover][pushover-url] to send notifications to any mobile device when a known MAC-address reconnects to the WiFi network.

 The main purpose of the project is to illustrate, given that you have access to the hardware, how easy it is to set up such monitoring, and that the course material in CS50 is more than enough to do so.


## Prerequisites

Below are the required software and how to install it. NB: If you have multiple projects on your Pi and run into dependency issues, you might want to use `pip` and a dedicated Python virtualenv for your setup.
* Raspberry Pi with Raspbian OS and WiFi card
* pyhton3
* sqlite3
* [gunicorn3][gunicorn3-url], `sudo apt install gunicorn3`
* [flask][flask-url],  `sudo apt install python3-flask`
* [svgwrite][svgwrite-url], `sudo apt install python3-svgwrite`
* [PyYAML][pyyaml-url], `sudo apt install python3-yaml`
* [Pushover account][pushover-url], register manually on site

## Getting Started

1. Setup up and connect your Raspbian Raspberry Pi to your WiFi network
2. Download the macreport software directly from the author
3. Check and install the additional prerequisites above
4. Go through the configuration section steps 1-4 below
5. Test your setup by running the following command from the project root folder
   - `export MACREPORT_ROOT=<full path to project_root>`
   - `python3 macreport/scanner.py --db data/database/mac.db`</br>
   This should print some debug output to the stdout and update the database with currently connected devices
6. Fix the final configuration, step 5 and onward, to schedule the software and get the Flask server up and running.

## Configuration
1. In the `config/macconfig.yaml`-file, update the entries listed below according to your needs. Note that other entries do not need to be updated.
   - `ip_range: 192.168.XX.X/24` - The network the Raspberry Pi is connected to
   - `macs_to_log:` - List of IPv4 MAC addresses to store with timestamp in the SQLITE database
   - `pusher:` - Settings for when to send a push notice, sub-entries are
      - `start_hour`: the time to start send notices (do not send any notice prior to this hour)
      - `min_hour_away`: Minimum number of hours time the MAC-address has to be offline before a notice is sent when it reconnects
   - `macs_to_push:` - List of IPv4 MAC address that may trigger a push notice
   - `person_to_macs:` - Mapping between one name and one (1) or more MAC addresses. The name is used in graphics and in any push notices sent
   - `time:` - Time settings for the generated graphics, sub-entries are
      - `hours2include` - Number of hour to list in textual macreport display
      - `binsize_minutes` - The binsize aggregate for data in the SQLITE database
2. In the file `macreport/sqlite/sqlite_queries.py`, update the entry `initialize_maclookup` to match your devices that will connect. The fields to enter are,
   - `macaddr`: The IPv4 MAC-address, used to identify connected devices
   - `hostname,`: The device hostname, complements macaddr when identifying connected devices
   - `macname`: Any name, only used to remember what device the first two columns refer two
3. `export MACREPORT_ROOT=<path to project root>`
4. Run `python3 macreport/reset_db.py --db data/database/mac.db` to initialize the database.
5. Schedule the following jobs, here using crontab syntax. The flag `-p` for production is used to suppress debug/info logging from the scripts, remove it when looking for errors.
- `*/5 *  *   *   *     <path-to-python3> <path-to-macreport/scanner.py> --db <path to database> -p >> <path-to-crontab-log> 2>&1;`
- `*/6 *  *   *   *     <path-to-python3> <path-to-macreport/push-if-home.py> --db <path to database> -p >> <path-to-crontab-log> 2>&1;`
6. Start Flask via (or set a reboot command in crontab),
   - `gunicorn3 "flaskr:create_app()" -b 0.0.0.0:5050 --timeout 3600 --daemon`
   The timeout is required for the Pi to finish when regenerating all svg images, i.e., all months in the database.
7. Update the file `macreport/pushers/pushover.py` with proper `token` and `user` from the Pushover account created in section "[Getting started](#getting-started)"
8. Check that the use cases in the section below are present


## Use Cases
Three use cases exist, two use cases from the Flask server and one use case when sending a push notice.
### Flask text view use case
The flask server will in the text view fetch and bin the 5-minute interval in the database into the configured time span, default 30 minutes, and display them as a simple list. This binning is made to handle the fact that not all devices will reply to every call from the `nmap` probe, but during a 30 minute sequence it is reasonable they will reply at least once. See

![alt text](https://github.com/marev711/macreport/blob/master/images/macreport-text-view.png?raw=true)

The image contain real data but names have been temporarily anonymized for the image. Short links `12h`, `24h`, etc.. can be used to filter how many hours from the start date to display. These links also adds a query variable `date=YYYYmmddHHMM` to the url which can be edited manually to display any date from the database. The text view is mainly intended for debugging.

### Flask svg view use case
The flask server will in the svg view fetch and bin the 5-minute intervals in the database into the configured time span, then write an SVG overview image for each month in the database. As above, the binning is made to handle the fact that not all devices will reply to every call from `nmap`, but during a 30 minute sequence it is reasonable they will reply at least once. See,

![alt text](https://github.com/marev711/macreport/blob/master/images/macreport-svg-view.png?raw=true)

Only the current month is regenerated unless the "Regenerate all images" is requested, other months are cached and reused. Color legend for this view is:

* green: device was detected at least once during the bin interval
* gray: device was not detected during the bin interval
* white: future (not yet in database)
* bold blue: Mondays

Note the intermittent gray boxes for PersonD the 27:th and 28:th, this is an iPhone that was present but did not reply to `nmap` during this time interval.

### Pushover notice
The pushover script is run separately and will read the database and when a known MAC-address reconnects, send a push notice to via the [Pushover][pushover-url]-service. For an example of the push notice received, see,

![alt text](https://github.com/marev711/macreport/blob/master/images/macreport-pushover-notice.png?raw=true)

The criterias for sending the push notice are,
* Push notices are not sent prior to a configurable time, e.g., not sent prior to 13:00 hours
* The MAC-address has to be absent at least a configurable amount of time, e.g., 4 hours

The main purpose of the script is to check whether family members have returned home in the afternoon.

## Roadmap
Desired features in near term are:
* Add an audio queue to a speaker connected to the Raspberry Pi when a known MAC address reconnects to the WiFi network. This should visualize ("audiolize") the ongoing monitoring and remind the persons monitored that digital monitoring is ubiquitous in their everyday life.
* Improve robustness for detecting devices connected to the WiFi network. The current `nmap` probe solution does not always work, e.g., an iPhone in sleep mode or with locked screen does not always reply.
* Add entry to select starting date in the text view use case.


## Contributing

Contributing is not possible as the project is intended  only as an assignment for the [CS50][cs50-url] course.


## License

Distributed under the MIT License. See `LICENSE` for more information.

## Contact

Send a PM to `d9023q4ladl` on [Reddit](https://reddit.com)

## Acknowledgements

* This README adapted from: [Best README Template by Othneil Drew](https://github.com/othneildrew/Best-README-Template)
* Flask CSS adapted from: [Official documentation example](https://flask.palletsprojects.com/en/1.1.x/tutorial/static/)
* Surveillance camera logotype from [Pixabay](https://pixabay.com)
* Various examples and assignments from the CS50 course have been reused.



<!-- MARKDOWN LINKS & IMAGES -->
<!-- https://www.markdownguide.org/basic-syntax/#reference-style-links -->
[issues-url]: https://github.com/marev711/macreport/issues
[license-url]: https://github.com/marev711/macreport/blob/master/LICENSE
[pushover-url]: https://pushover.net
[gunicorn3-url]: https://docs.gunicorn.org/en/stable/index.html
[flask-url]: https://palletsprojects.com/p/flask/
[svgwrite-url]: https://pypi.org/project/svgwrite/
[cs50-url]: https://online-learning.harvard.edu/course/cs50-introduction-computer-science
[pyyaml-url]: https://pyyaml.org/
