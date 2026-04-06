# Fix: World of Warcraft In-Game Shop Black Screen on Linux

When opening the in-game shop in World of Warcraft on Linux, the embedded browser may render as a black screen. This occurs because the Chromium Embedded Framework (CEF) sandbox strictly rejects the Linux bridging drive (usually `Z:`) used by Wine/Proton to map paths when Battle.net and the game are installed in different prefixes.

The solution is to map a native local drive letter within Battle.net to bypass the restriction.

Choose either the **Manual Configuration** method or the **Automated Script** method below.

---

## Method 1: Manual Configuration

Follow these steps to manually map the correct path.

**1. Identify Paths**
You need the absolute paths to two locations:
- **Battle.net Prefix:** The active Wine prefix directory where Battle.net is running. You need the path to its `dosdevices` folder.
- **WoW Directory:** The folder containing the `_retail_` directory.

**2. Map the Drive**
Create a new drive letter (`W:`) inside the Battle.net `dosdevices` directory pointing to the physical WoW directory.
```bash
ln -snf "/path/to/World of Warcraft" "/path/to/battlenet_prefix/dosdevices/w:"
```

**3. Clear the Cache**
Delete the `UTILS` folder inside the WoW directory to purge the broken sandbox configuration.
```bash
rm -rf "/path/to/World of Warcraft/_retail_/UTILS"
```

**4. Update Battle.net**
1. Close Battle.net completely.
2. Relaunch Battle.net.
3. Click **Locate the game**.
4. Select the newly mapped `(W:)` drive shown in the file tree.

---

## Method 2: Automated Script

If you prefer an automated approach, this script discovers your active prefixes, maps the drive, and clears the cache for you.

**1. Create the Script File**
Create a file named `wow_shop_fix.sh` in your home directory and paste the following code into it:

```bash
#!/usr/bin/env bash
# wow_shop_fix.sh - Automates the W: drive fix for the World of Warcraft Cash Shop Black Screen bug on Linux

echo "========================================================="
echo "   World of Warcraft - In-Game Shop Black Screen Fix     "
echo "========================================================="
echo "Scanning for Battle.net and World of Warcraft locations..."
echo "This might take a few seconds..."

# 1. Find Battle.net prefixes
declare -a bnet_prefixes
while IFS= read -r pfx; do
    if [[ -d "$pfx" ]]; then
        bnet_prefixes+=("$pfx")
    fi
done < <(find ~/.local/share/Steam/steamapps/compatdata ~/Games ~/.wine -maxdepth 6 -type d -name "dosdevices" 2>/dev/null)

declare -a valid_bnet
for pfx in "${bnet_prefixes[@]}"; do
    if [[ -f "$pfx/../drive_c/ProgramData/Battle.net/Agent/product.db" ]] || \
       [[ -f "$pfx/../drive_c/users/steamuser/AppData/Roaming/Battle.net/Battle.net.config" ]] || \
       [[ -d "$pfx/../drive_c/Program Files (x86)/Battle.net" ]]; then
       valid_bnet+=("$pfx")
    fi
done

if [[ ${#valid_bnet[@]} -eq 0 ]]; then
    echo "ERROR: Could not automatically detect any Battle.net prefixes."
    echo "Please launch Battle.net at least once or verify its installation location."
    exit 1
fi

selected_bnet=""
if [[ ${#valid_bnet[@]} -eq 1 ]]; then
    selected_bnet="${valid_bnet[0]}"
    echo "[+] Found Battle.net prefix: $selected_bnet"
else
    echo "---------------------------------------------------------"
    echo "Multiple Battle.net prefixes found. Please select the ACTIVE one:"
    select opt in "${valid_bnet[@]}"; do
        if [[ -n "$opt" ]]; then
            selected_bnet="$opt"
            break
        else
            echo "Invalid selection."
        fi
    done
fi

# 2. Find WoW installations
declare -a wow_installs
while IFS= read -r wow; do
    if [[ -d "$wow" ]]; then
        wow_installs+=("$wow")
    fi
done < <(find ~/.local/share/Steam/steamapps/compatdata ~/Games ~/.wine -maxdepth 8 -type d -name "World of Warcraft" 2>/dev/null)

if [[ ${#wow_installs[@]} -eq 0 ]]; then
    echo "ERROR: Could not automatically detect a World of Warcraft installation."
    exit 1
fi

selected_wow=""
if [[ ${#wow_installs[@]} -eq 1 ]]; then
    selected_wow="${wow_installs[0]}"
    echo "[+] Found WoW installation: $selected_wow"
else
    echo "---------------------------------------------------------"
    echo "Multiple WoW installations found. Please select the CORRECT one:"
    select opt in "${wow_installs[@]}"; do
        if [[ -n "$opt" ]]; then
            selected_wow="$opt"
            break
        else
            echo "Invalid selection."
        fi
    done
fi

# 3. Create the drive mapping
echo "---------------------------------------------------------"
echo "Injecting native drive mapping..."
target_drive="w:"
bnet_dosdevices="$selected_bnet"

if [[ -e "$bnet_dosdevices/$target_drive" ]]; then
    echo " > Replacing existing $target_drive mapping..."
    rm -f "$bnet_dosdevices/$target_drive"
fi

ln -snf "$selected_wow" "$bnet_dosdevices/$target_drive"

if [[ -e "$bnet_dosdevices/$target_drive" ]]; then
    echo "[+] Successfully mapped (W:) drive to your WoW installation."
else
    echo "ERROR: Failed to create drive mapping symlink. Verify permissions."
    exit 1
fi

# 4. Clear the UTILS browser cache
utils_folder="$selected_wow/_retail_/UTILS"
if [[ -d "$utils_folder" ]]; then
    echo "[+] Clearing corrupted browser sandbox configuration..."
    rm -rf "$utils_folder"
fi

# 5. Final Instructions
echo "========================================================="
echo "   [ SUCCESS ] Fix successfully applied to prefix!       "
echo "========================================================="
echo "Next Steps:"
echo "1. Completely CLOSE and REOPEN your Battle.net app."
echo "2. Go to the World of Warcraft tab and click 'Locate the game'."
echo "3. In that folder UI, select the new native (W:) drive."
echo "4. Open the game! Your shop browser will now securely render."
echo "========================================================="
```

**2. Make Executable and Run**
Open your terminal and execute the script:
```bash
chmod +x ~/wow_shop_fix.sh
~/wow_shop_fix.sh
```

**3. Update Battle.net**
1. Close Battle.net completely.
2. Relaunch Battle.net.
3. Click **Locate the game**.
4. Select the newly mapped `(W:)` drive.
