#Tool Property Of Maim
#!/bin/bash
clear

# Function to read device data from the file
function read_devfile {
  declare -g code_type code_length last_value
  if [[ -f "${devfile}" ]]; then
    while IFS='=' read -r key value; do
      case "${key}" in
        code_type) code_type="${value}" ;;
        code_length) code_length="${value}" ;;
        last_value) last_value="${value}" ;;
      esac
    done < "${devfile}"
  fi
}

# Function to write device data to the file
function write_devfile {
  {
    echo "code_type=${code_type}"
    echo "code_length=${code_length}"
    echo "last_value=${last_value}"
  } > "${devfile}"
}

# Function to convert a number to a code string based on the character set
function number_to_code {
  local num=$1
  local length=$2
  local base=${#charset}
  local code=''
  while [[ ${#code} -lt ${length} ]]; do
    local index=$(( num % base ))
    code="${charset:index:1}${code}"
    num=$(( num / base ))
  done
  echo "${code}"
}

# Get the current device name
devices=( $(fastboot devices) )
device="${devices[0]}"
printf "Current device: %s\n" "$device"
# This part of the code creates a file name based on your device name
devfile="./${device}.dat"

# If device data file doesn't exist, prompt for code type and length
if [[ ! -f "${devfile}" ]]; then
  while true; do
    read -r -p "Does the device use numeric or alphanumeric codes? (n/a): " code_type
    if [[ "${code_type}" =~ ^(n|a|numeric|alphanumeric)$ ]]; then
      [[ "${code_type}" == 'n' ]] && code_type='numeric'
      [[ "${code_type}" == 'a' ]] && code_type='alphanumeric'
      break
    else
      echo "Please enter 'n' for numeric or 'a' for alphanumeric."
    fi
  done
  read -r -p "Enter code length: " code_length
  last_value=0
  write_devfile
else
  read_devfile
fi

# Set character set based on code type
if [[ "${code_type}" == "numeric" ]]; then
  charset='0123456789'
elif [[ "${code_type}" == "alphanumeric" ]]; then
  charset='0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz'
else
  echo "Invalid code type: ${code_type}"
  exit 1
fi

base=${#charset}

# Trap to save data on exit
trap 'write_devfile; exit' EXIT SIGINT

while true; do
  code=$(number_to_code ${last_value} ${code_length})

  # Try to unlock the device
  output=($(fastboot oem unlock "${code}" 2>&1))

  # Check if the unlock was successful
  if ! echo "${output}" | grep -iqE 'fail(ed|ure)?'; then
    echo -e "\nYour unlock code is: ${code}"
    break
  fi

  # Display the current attempt
  echo -ne "Trying code: ${code}\r"

  (( last_value++ ))
  write_devfile
done
