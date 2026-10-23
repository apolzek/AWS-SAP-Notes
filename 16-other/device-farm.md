# AWS Device Farm

- Device Farm is service which provides managed Web and Mobile application testing
- We can test an application on a fleet of real browser and devices
- Device Farm provides access to real devices: phones, tablets, different languages, sizes and operating systems
- We can use build in or supported automated testing frameworks from which we can receive reports detailing testing output
- It also supports remote connection to devices for issue reproduction and testing
- We can define tests using testing tools such as Explorer (Android), Fuzz (Android, iOS), Web app Tests, Appium, Calabash
- We can configure which devices we want to use for testing (huge selection is available). We can configure device state, additional apps, radio states, locations, etc.
- Two main offerings:
    - **Mobile device testing**: real Android and iOS phones/tablets in AWS data centers (us-west-2 region); upload app (APK/IPA) + test package
    - **Desktop browser testing**: Selenium tests on desktop browsers (Chrome, Firefox, Edge) hosted by AWS, run against a managed Selenium Grid endpoint
- Test frameworks: Appium (Java, Python, Node.js, Ruby), Espresso, XCTest/XCUITest, built-in fuzz and explorer tests; Calabash and UI Automation are legacy/removed
- Results: logs, screenshots, video recordings, performance data (CPU, memory, threads) per device
- **Remote access**: interact with a real device from the browser (swipe, gesture) for manual testing/reproduction
- **Private devices**: dedicated devices reserved for our account (specific models/OS, can be used with VPC connectivity - **VPC-ENI** to reach private test endpoints)
- Integration with CI/CD: CodePipeline test action, Jenkins, Gradle plugin, CLI/SDK
- Pricing: pay per device minute, or unmetered plans (flat monthly fee per device slot)
- Exam: "test mobile app on many real devices/OS versions in parallel, automated, from CI/CD" -> Device Farm
