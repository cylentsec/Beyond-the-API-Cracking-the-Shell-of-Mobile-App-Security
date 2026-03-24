# Examples of automating the OWASP MASTG

Add shell functions to your `~/.zshrc` or `~/.bashrc` file

## List all non-Apple applications:

```bash
function csec-ios-list-apps() {
    local SEARCH_TERM="$1"
    ipsw idev apps ls | awk '{print $1, $2, $3, $4}' | grep -i "$SEARCH_TERM"
}
```

## Set app package name (com.example.app) to variable:

```bash
function csec-ios-apn-to-variable() {
    local APN="$1"
    # 1. Run the command and capture the output into a variable
    search_result=$(ipsw idev apps ls | awk '{print $1, $2, $3, $4}' | grep -i "$APN")
    # 2. Count the number of non-empty lines in the result
    match_count=$(echo "$search_result" | grep -c .)
    # 3. If (and only if) there is exactly 1 match, extract the text inside ()
    if [ "$match_count" -eq 1 ]; then
        # -F'[()]' tells awk to split the line by '(' and ')'
        # print $2 prints the text inside the first pair of parentheses
        APP_ID=$(echo "$search_result" | awk -F'[()]' '{print $2}')
        APP_PATH=$(echo "$search_result" | awk '{print $4}')
    fi
}
```

## MASTG-TEST-0069: Testing App Permissions

Examine the permissions and determine if the permissions are reasonable for the purpose of the application. If you find anything questionable you can ask the customer about it. Consider that permissions are dictated by business requirements and you must use reasonable judgement here.

```bash
function csec-ios-MASTG-TEST-0069() {
    objection -n "$APP_ID" -s -p run "ios plist cat Info.plist" | grep -E "Usage|Permission"
}
```

## MASTG-TEST-0052: Testing Local Data Storage

**NOTE:** For MASVS L1 compliance, it is sufficient to store data unencrypted in the application's internal storage directory (sandbox). For L2 compliance, additional encryption is required using cryptographic keys securely managed in the iOS KeyChain\. This includes using envelope encryption (DEK+KEK) or equivalent methods.

#### **Step 1:** Trigger all possible functionality in the application
* Use a tool like Xcode and an iOS simulator to test the app thoroughly.
* Click everywhere possible to ensure data generation and modification.
* Test various scenarios, such as login/logout, adding/removing items from lists, etc.

#### **Step 2:** Find and dump sqlite files

```bash
function csec-ios-enum-app-base-path() {
    objection -g "$APP_ID" run "ios bundles list_bundles --full-path"
}
```

This will show you the app's bundle directory, documents directory, and other relevant paths.
Then, from a SSH terminal on the device, for each BASE path, run `./dump_sqlite.sh <path>`. The contents of the `dump_sqlite.sh` file is:

```bash
#!/var/jb/usr/bin/bash

SEARCH_PATH=$1
DUMP_PATH=sqlite_dump_$(date +%Y%m%d_%H%M%S).txt
find "$SEARCH_PATH" -type f -name "*.sqlite" | while read -r db_file; do
    # Dump database structure and data
    echo "$db_file" >> "$DUMP_PATH" && sqlite3 "$db_file" ".dump" >> "$DUMP_PATH"
done
less "$DUMP_PATH"
```

To determine if the Keychain configuration is secure within an app, follow these guidelines:
1. Verify that sensitive data is stored with proper protection: Check that sensitive assets like passwords or encryption keys are being saved using a Keychain attribute such as `kSecAttrAccessibleWhenPasscodeSetThisDeviceOnly`, which requires both passcode and Touch ID authentication.
2. Ensure access to the items is properly restricted: Verify that only authorized apps signed by the same developer have access to shared data through the use of an [access group](https://support.apple.com/en-lb/guide/security/secb0694df1a/web#:~:text=Rather%20than%20limiting%20access%20to,use%20distinct%20keys%20and%20functions.&text=Apps%20that%20use%20background%20refresh,t%20included%20in%20escrow%20keybags)\.
3. Make sure Keychain protection aligns with app's sensitivity level: Confirm that sensitive information is properly encrypted and protected based on its sensitivity, such as using the `kSecAttrAccessibleWhenPasscodeSetThisDeviceOnly` attribute for high-risk data.

To check these conditions in practice:
1. Inspect the Keychain configuration within an instance of Xcode\.
2. Look out for attributes like `kSecAttrAccessibleWhenPasscodeSetThisDeviceOnly`, and ensure they're used correctly based on sensitivity levels.
3. Check entitlements, such as `Keychain-access-groups` or `application-identifier`, to verify that access control is properly implemented.
Dump the keychain:

```bash
objection --gadget "$APP_ID" run "ios keychain dump"
```

#### Step 3: Dump info.plist content

```bash
objection --gadget "$APP_ID" run "ios plist cat Info.plist"
```

#### Step 4: Test for sensitive data in nsuserdefaults

```bash
objection -g "$APP_ID" run "ios nsuserdefaults get"
```

#### Step 5: Check Core Data for sensitive information

This data is located in the app DocumentDirectory\. In the  SSH terminal, run:

```bash
sqlite3 <path>/CoreData.sqlite ".dump"
```

#### Step 6: Check for additional plist files and check content

from a SSH terminal on the device, for each BASE path, run `./dump_plist.sh <path>`\. The contents of the `dump_plist.sh` file is:

```bash
#!/var/jb/usr/bin/bash

SEARCH_PATH=$1
DUMP_PATH=plist_dump_$(date +%Y%m%d_%H%M%S).txt
find "$SEARCH_PATH" -type f -name "*.plist" | while read -r plist_file; do
    # Dump plist data
    echo "$plist_file" >> "$DUMP_PATH" && plutil "$plist_file" >> "$DUMP_PATH"
done
less "$DUMP_PATH"
```
