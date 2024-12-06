#!/bin/bash

# Prompt user for the target directory (default to the user's home directory if none is provided)
read -p "Please enter the target directory (default: ~/hashproject): " TARGET_DIR
TARGET_DIR=${TARGET_DIR:-"$HOME/hashproject"}  # Default to ~/hashproject if not provided

# Check if the target directory exists, if not, create it
if [ ! -d "$TARGET_DIR" ]; then
    echo "Directory does not exist. Creating $TARGET_DIR..."
    mkdir -p "$TARGET_DIR"
fi

# Function to calculate the hash of a file
calculate_file_hash() {
    local filepath=$1
    sha512sum "$filepath" | awk '{print $1}'
}

# Function to erase baseline if it already exists
erase_baseline_if_exists() {
    if [ -f "$TARGET_DIR/baseline.txt" ]; then
        rm "$TARGET_DIR/baseline.txt"
    fi
}

# Prompt the user for action
echo ""
echo "What would you like to do?"
echo ""
echo "    A) Collect new Baseline?"
echo "    B) Begin monitoring files with saved Baseline?"
echo ""
read -p "Please enter 'A' or 'B': " response
echo ""

if [ "${response^^}" == "A" ]; then
    # Collect new baseline
    erase_baseline_if_exists

    # Loop through all files in the target directory and calculate hashes
    for file in "$TARGET_DIR"/*; do
        if [ -f "$file" ]; then
            hash=$(calculate_file_hash "$file")
            echo "$file|$hash" >> "$TARGET_DIR/baseline.txt"
        fi
    done
    echo "Baseline collected and saved to $TARGET_DIR/baseline.txt"

elif [ "${response^^}" == "B" ]; then
    declare -A file_hash_dictionary

    # Load baseline into dictionary
    if [ -f "$TARGET_DIR/baseline.txt" ]; then
        while IFS="|" read -r filepath filehash; do
            file_hash_dictionary["$filepath"]=$filehash
        done < "$TARGET_DIR/baseline.txt"
    else
        echo "No baseline found. Please create a baseline first."
        exit 1
    fi

    echo "Monitoring files in $TARGET_DIR..."
    while true; do
        sleep 1

        # Check for changes, additions, and deletions
        for file in "$TARGET_DIR"/*; do
            if [ -f "$file" ]; then
                hash=$(calculate_file_hash "$file")
               
                if [[ -z "${file_hash_dictionary[$file]}" ]]; then
                    echo -e "\e[32m$file has been created!\e[0m"
                elif [[ "${file_hash_dictionary[$file]}" != "$hash" ]]; then
                    echo -e "\e[33m$file has changed!!!\e[0m"
                fi
                file_hash_dictionary["$file"]=$hash
            fi
        done

        # Check for deleted files
        for key in "${!file_hash_dictionary[@]}"; do
            if [ ! -f "$key" ]; then
                echo -e "\e[41;30m$key has been deleted!\e[0m"
                unset file_hash_dictionary["$key"]
            fi
        done
    done
else
    echo "Invalid option. Please enter 'A' or 'B'."
    exit 1
fi
