# `google-chrome` + custom plugin

This took a lot of work to get right, and i am no longer using it in my private project so here you go. This is tested on ubuntu server with a tightvnc server running using `localhost:1`. This fully installs google-chrome along with a custom plugin.

These scripts were ran using vagrant, cut-pasted:

    chrome.vm.provision :shell, :path => "provision/ubuntu-chrome-google.sh"
    chrome.vm.provision :shell, :path => "provision/ubuntu-project-bot-chrome-plugin.sh", :args => "/opt/shared/bot-chrome-plugin"

`/opt/shared/bot-chrome-plugin/` would be your plugin source folder.

## `ubuntu-chrome-google.sh`

Automatically installs `google-chrome` from Google's repositories, and makes sure it does not pop up with its "first run" dialog the first time you run it.

    #!/bin/sh
    
    # ----------------------------
    # google chrome
    # ----------------------------
    
    wget -q -O - https://dl-ssl.google.com/linux/linux_signing_key.pub | sudo apt-key add -
    sudo cat <<EOF >> /etc/apt/sources.list
    deb http://dl.google.com/linux/chrome/deb/ stable main
    EOF
    
    sudo apt-get -y update
    sudo apt-get -y install google-chrome-stable
    export DISPLAY=localhost:1
    mkdir -p 'chrome/First Run'
    google-chrome --user-data-dir=chrome --ingocnito &

## `ubuntu-project-bot-chrome-plugin.sh`

Waits for chrome to create its folder structure, then kills it, packs the extension and inserts a special plugin manifest file into the user's `Preferences` file.

    #!/bin/sh
    
    # ----------------------------
    # bot-chrome-plugin
    # ----------------------------
    #
    # trivia:
    #
    # the "extension id" is the sha256 of an rsa public key, encoded in an alternative base 16 using a-p as opposed to 0-f.
    # to get the id programmatically, you can use this:
    #     openssl rsa -pubout -outform DER < bot-chrome-plugin.pem | sha256sum | head -c32 | tr 0-9a-f a-p
    # this also means that the extension id is not an really an id of the extension, but an update authentication mechanism.
    # meaning we can use any bullshit extension id, and as long as it is in the alternative base16 ( a-p vs 0-f ) it'll work.
    # ( see https://code.google.com/p/chrome-browser/source/browse/trunk/src/chrome/common/extensions/extension.cc )
    
    # fix for "error while loading shared libraries: libudev.so.0"
    # ( not tested if this error also happens during a vagrant installation, but this fixed a development environment )
    ln -s /lib/x86_64-linux-gnu/libudev.so.1 /lib/x86_64-linux-gnu/libudev.so.0
    
    export DISPLAY=localhost:1
    
    while [ ! -f ./chrome/Default/Preferences ]
    do
      sleep 1
    done
    
    sudo killall chrome
    
    # generate .crx and .pem file, they'll be at $1.crx and $1.pem
    # not that it really matters, the .crx is actually just a .zip file and them .pem file is only for the extension id.
    google-chrome --pack-extension=$1 --no-message-box  --user-data-dir=chrome
    
    # now it is up to us to edit /home/vagrant/chrome/Default/Preferences
    # find-replace one of the default plugin lines with our config + the line we would otherwise replace:
    
    sed -i 's#"mfehgcgbbipciphmccgaenjidiccnmng#\
            "khcecimilnhmlkfbnbkhipgnhkpjnecd": {\
                "active_permissions": {\
                   "api": [ "activeTab", "tabs", "alarms", "background" ],\
                   "explicit_host": [ "<all_urls>", "chrome://favicon/*" ],\
                   "scriptable_host": [ "<all_urls>" ]\
                },\
                "creation_flags": 38,\
                "from_bookmark": false,\
                "from_webstore": true,\
                "granted_permissions": {\
                   "api": [ "activeTab", "tabs", "alarms" ],\
                       "explicit_host": [ "<all_urls>", "chrome://favicon/*" ],\
                   "scriptable_host": [ "<all_urls>" ]\
                },\
                "initial_keybindings_set": true,\
                "install_time": "13036541413460503",\
                "location": 4,\
                "newAllowFileAccess": true,\
                "path": "/opt/shared/bot-chrome-plugin",\
                "state": 1,\
                "was_installed_by_default": true\
             },\
            "mfehgcgbbipciphmccgaenjidiccnmng#' ./chrome/Default/Preferences
    
    nohup google-chrome --user-data-dir=chrome --ingocnito &

Most important from this is that to have a persiting background script, you'll need to add it as a permission and mark it as installed from the webstore. The key that you use does not matter as you can read in the trivia bit.
