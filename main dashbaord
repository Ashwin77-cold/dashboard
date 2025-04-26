import os
import time
import re
from datetime import datetime
from flask import Flask, render_template_string, send_file, make_response
import pandas as pd

app = Flask(__name__)

# Define the directories for monitoring
DIRECTORIES = [
    (r"A:\logs", "JPF1M183", "ST_1"),
    (r"B:\logs", "OMDLAS1220", "ST_2"),
    (r"E:\logs", "JPF1A184", "ST_3"),
    (r"F:\logs", "JPF1N117", "ST_4"),
    (r"G:\logs", "SHSS083", "ST_5"),
    (r"H:\logs", "JR0546", "ST_6"),
#     (r"I:\logs", "JPF1V170", "ST_7"),
]

# Define starting times
STARTING_TIMES = {
    "SCENARIO_SUPER_1_C": "09:17:00",
    "SCENARIO_SUPER_2_C": "09:30:00",
    "SCENARIO_SUPER_3_C": "10:00:00",
    "SCENARIO_SUPER_4_C": "10:30:00",
    "SCENARIO_SUPER_1_D": "09:18:00",
    "SCENARIO_SUPER_2_D": "09:31:00",
    "SCENARIO_SUPER_3_D": "10:01:00",
    "SCENARIO_SUPER_4_D": "10:31:00"
   }

def list_files(directory):
    try:
        files = os.listdir(directory)
        return [f for f in files if os.path.isfile(os.path.join(directory, f))]
    except Exception:
        return []

def filter_files(files, keywords):
    return [file for file in files if any(keyword in file for keyword in keywords)]

def extract_super_name(file_name):
    match = re.search(r'(scenario_super_\d+_[A-Z]?)', file_name, re.IGNORECASE)
    return match.group(1).upper() if match else "UNKNOWN"

def determine_status(content_lines):
    max_loss_found = False
    max_rentry_reached_found = False
    running_found = False
    trade_not_taken_found = False

    for line in reversed(content_lines):
        if "max Loss Reached" in line:
            max_loss_found = True
        if "total_rentry_limit reached" in line:
            max_rentry_reached_found = True
        if "LTP Response from socket" in line:
            running_found = True
        if "Waiting for market to open" in line:
            trade_not_taken_found = True

    if max_loss_found and max_rentry_reached_found:
        return "MAX RENTRY REACHED"
    if max_loss_found:
        return "MAX LOSS"
    if max_rentry_reached_found:
        return "MAX RENTRY REACHED"
    if running_found:
        return "RUNNING"
    if trade_not_taken_found:
        return "TRADE NOT TAKEN"

    return "UNKNOWN"

def extract_total_pnl(content_lines):
    for line in reversed(content_lines):
        if "TotalPNL :" in line:
            match = re.search(r'TotalPNL\s*:\s*([\d.-]+)', line)
            if match:
                return float(match.group(1))
    return None

def read_file(file_path):
    if not os.path.exists(file_path):
        return "UNKNOWN", None
    try:
        with open(file_path, 'r') as file:
            content_lines = file.readlines()
        status = determine_status(content_lines)
        total_pnl = extract_total_pnl(content_lines)
        return status, total_pnl
    except Exception:
        return "UNKNOWN", None

def get_latest_date_folder(base_path):
    folders = [d for d in os.listdir(base_path) if os.path.isdir(os.path.join(base_path, d))]
    return max(folders, key=lambda d: datetime.strptime(d, '%Y%m%d')) if folders else None

def get_client_path(base_path, client_name):
    client_path = os.path.join(base_path, client_name)
    return client_path if os.path.exists(client_path) else None

def get_index_folder(date_path):
    subfolders = [d for d in os.listdir(date_path) if os.path.isdir(os.path.join(date_path, d))]
    return os.path.join(date_path, subfolders[0]) if subfolders else None

def read_update_time_files(status_folder):
    update_times = {}
    for file_name in list_files(status_folder):
        super_name = extract_super_name(file_name)
        if super_name:
            file_path = os.path.join(status_folder, file_name)
            try:
                last_modified_time = os.path.getmtime(file_path)
                update_times[super_name] = last_modified_time
            except Exception:
                pass
    return update_times

def get_current_time():
    return time.time()

def check_for_abrupt_closure(update_times, log_statuses):
    current_time = get_current_time()
    for super_name, update_time in update_times.items():
        if (current_time - update_time) > 60:
            log_status = log_statuses.get(super_name, "UNKNOWN")
            if log_status not in ["MAX LOSS", "TRADE NOT TAKEN","MAX RENTRY REACHED"]:
                log_statuses[super_name] = "ABRUPT CLOSURE"
    return log_statuses

def print_status():
    combined_statuses = {}

    for base_path, client_name, system_name in DIRECTORIES:
        client_path = get_client_path(base_path, client_name)
        if not client_path:
            continue

        latest_date_folder = get_latest_date_folder(client_path)
        if not latest_date_folder:
            continue

        date_path = os.path.join(client_path, latest_date_folder)
        index_path = get_index_folder(date_path)
        if not index_path:
            continue

        status_folder = os.path.join(index_path, "code_running_status")
        file_names = list_files(index_path)

        keywords = [f"scenario_super_{i}" for i in range(1, 5)]
        filtered_files = filter_files(file_names, keywords)

        statuses = {}
        total_pnls = {}
        last_modified_times = {}  # Initialize the dictionary here

        for file_name in filtered_files:
            super_name = extract_super_name(file_name)
            status, total_pnl = read_file(os.path.join(index_path, file_name))
            statuses[super_name] = status
            total_pnls[super_name] = total_pnl

            # Get the last modified time
            file_path = os.path.join(index_path, file_name)
            last_modified_time = os.path.getmtime(file_path)
            last_modified_times[super_name] = datetime.fromtimestamp(last_modified_time).strftime('%Y-%m-%d %H:%M:%S')

        update_times = read_update_time_files(status_folder)
        statuses = check_for_abrupt_closure(update_times, statuses)

        combined_statuses[system_name] = (list(statuses.keys()), list(statuses.values()), total_pnls, last_modified_times, os.path.basename(index_path).upper())

    return combined_statuses


@app.route('/csv/<index_name>/<super_name>/<system_name>')
def display_csv(index_name, super_name, system_name):
    # Ensure everything is lowercase
    index_name_lower = index_name.lower()
    super_name_lower = super_name.lower()

    # Construct the file name
    file_name = f"{index_name_lower}_{super_name_lower}_{system_name}.csv"
    file_path = os.path.join("D:\\DASH", file_name)

    # Check if the file exists
    if os.path.exists(file_path):
        # Read the CSV file into a DataFrame
        df = pd.read_csv(file_path)

        # Generate styled HTML
        html = f"""
        <!DOCTYPE html>
        <html lang='en'>
        <head>
            <meta charset='UTF-8'>
            <meta name='viewport' content='width=device-width, initial-scale=1.0'>
            <title>{system_name} CSV Data</title>
            <style>
                body {{
                    font-family: 'Arial', sans-serif;
                    background-color: #f8f9fa; /* Light gray background */
                    color: #333;
                    margin: 0; /* Remove default margin */
                    padding: 0; /* Remove default padding */
                }}
                .container {{
                    max-width: 100%; /* Full width */
                    margin: 0; /* Remove margin */
                    padding: 20px;
                    background: #ffffff; /* White background for the main content */
                    border-radius: 10px;
                    box-shadow: 0 4px 20px rgba(0, 0, 0, 0.1);
                }}
                h1 {{
                    text-align: center;
                    color: #007bff;
                    margin: 20px 0; /* Adjusted margin */
                }}
                table {{
                    width: 100%; /* Ensure table takes full width */
                    border-collapse: collapse;
                    margin-top: 0; /* Start right at the top */
                }}
                th, td {{
                    padding: 15px; /* Increased padding for better spacing */
                    text-align: center; /* Center align text */
                    border: 1px solid #ddd;
                    background-color: #ffffff; /* White background for all cells */
                }}
                th {{
                    background-color: #007bff; /* Header color */
                    color: white;
                    font-weight: bold;
                }}
                tr:nth-child(even) {{
                    background-color: #f2f2f2; /* Zebra stripes for readability */
                }}
                tr:hover {{
                    background-color: #e9ecef; /* Highlight on hover */
                }}
            </style>
        </head>
        <body>
            <div class='container'>
                <h1>{system_name}_{super_name}</h1>
                <table>
                    <tr>
                        {"".join(f"<th>{col}</th>" for col in df.columns)}  <!-- Dynamically create table headers -->
                    </tr>
        """

        # Populate table rows with CSV data
        for index, row in df.iterrows():
            html += "<tr>" + "".join(f"<td>{cell}</td>" for cell in row) + "</tr>"
        
        html += """
                </table>
            </div>
        </body>
        </html>
        """

        return render_template_string(html)
    else:
        return "File not found", 404

@app.route('/')
def dashboard():
    combined_statuses = print_status()
    current_time = datetime.now().strftime("%Y-%m-%d %H:%M:%S")

    systems_data = []
    for system_name, (super_names, status_values, total_pnls, last_modified_times, folder_name) in combined_statuses.items():
        statuses = dict(zip(super_names, status_values))  # Create a dictionary from lists

        # Sort super names based on the numeric part after "scenario_super_"
        sorted_super_names = sorted(
            super_names,
            key=lambda name: int(name.split('_')[2]) if len(name.split('_')) > 2 and name.split('_')[2].isdigit() else float('inf')
        )

        sorted_status_values = [statuses.get(name, 'N/A') for name in sorted_super_names]
        sorted_last_modified_times = {name: last_modified_times.get(name, 'N/A') for name in sorted_super_names}

        systems_data.append((system_name, sorted_super_names, sorted_status_values, total_pnls, sorted_last_modified_times, folder_name))

    # List to hold the reshaped data
    reshaped_data = []

    for system_name, (super_names, status_values, total_pnls, last_modified_times, folder_name) in combined_statuses.items():
        statuses = dict(zip(super_names, status_values))  # Create a dictionary from lists

        # Sort super names based on the numeric part after "scenario_super_"
        sorted_super_names = sorted(
            super_names,
            key=lambda name: int(name.split('_')[2]) if len(name.split('_')) > 2 and name.split('_')[2].isdigit() else float('inf')
        )

        sorted_status_values = [statuses.get(name, 'N/A') for name in sorted_super_names]

        # Use sorted_last_modified_times instead of last_modified_times directly
        for super_name in sorted_super_names:
            reshaped_data.append({
                "Picasso Name": super_name,
                "Status": statuses.get(super_name, 'N/A'),
                "System Name": system_name,
                "Total PNL": total_pnls.get(super_name, 'N/A'), 
                "Last Modified Time": sorted_last_modified_times.get(super_name, 'N/A'),  
                "Starting Time": STARTING_TIMES.get(super_name.strip().upper(), "N/A"),
                "Folder Name": folder_name
            })

    # Create DataFrame from the reshaped data
    df = pd.DataFrame(reshaped_data)

    # Create folder for saving CSV
    current_date = datetime.now().strftime("%Y-%m-%d")
    folder_path = os.path.join("D:\\DASH", current_date)
    if not os.path.exists(folder_path):
        os.makedirs(folder_path)

    csv_file_path = os.path.join(folder_path, 'dashboard_data_cd.csv')

    # Save the DataFrame to CSV, replacing the file every time
    df.to_csv(csv_file_path, mode='w', header=True, index=False)  # Overwrite the file

    # Return the rendered HTML
    html = f"""
    <!DOCTYPE html>
    <html lang='en'>
    <head>
        <meta charset='UTF-8'>
        <meta name='viewport' content='width=device-width, initial-scale=1.0'>
        <title>Status Dashboard</title>
        <meta http-equiv='refresh' content='30'>
        <style>
            body {{
                font-family: 'Arial', sans-serif;
                background-color: #ffffff;
                color: #333;
                margin: 0;
                padding: 0;
                display: flex;
                flex-direction: column;
                height: 100vh;
            }}
            h1 {{
                text-align: center;
                color: #001f3f;
                text-shadow: 2px 2px 4px rgba(0, 0, 0, 0.5);
                margin: 20px 0;
                font-size: 2.5em;
            }}
            .container {{
                display: flex;
                flex-wrap: wrap;
                justify-content: space-around;
                padding: 20px;
                flex-grow: 1;
                overflow-y: auto;
            }}
            .system {{
                background: #ffffff;
                border: 1px solid #ddd;
                border-radius: 12px;
                box-shadow: 0 8px 20px rgba(0, 0, 0, 0.2);
                margin: 15px;
                padding: 20px;
                width: calc(45% - 40px);
                transition: transform 0.3s, box-shadow 0.3s;
            }}
            .system:hover {{
                transform: scale(1.05);
                box-shadow: 0 12px 30px rgba(0, 0, 0, 0.3);
                background-color: #f8f9fa;
            }}
            .timestamp {{
                text-align: center;
                margin: 20px 0;
                font-size: 1.8em; 
                font-weight: bold; 
                color: #ffffff; 
                background-color: rgba(50, 125, 175, 1); 
                padding: 15px; 
                border-radius: 30px; 
                box-shadow: 0 4px 15px rgba(0, 0, 0, 5);
            }} 
            h2 {{
                color: #001f3f;
                margin: 10px 0;
            }}
            .table {{
                width: 100%; 
                border-collapse: collapse;
                margin-bottom: 20px;
            }}
            .th {{
                background-color: #327DAF;
                color: white;
                text-align: left;
                padding: 12px;
                border-radius: 8px 8px 0 0;
                font-weight: bold; 
            }}
            .td {{
                background: #f9f9f9;
                border: 1px solid #ddd;
                padding: 10px;
                transition: box-shadow 0.3s, transform 0.3s;
                font-weight: bold; 
                color: #333; 
            }}
            .td:hover {{
                box-shadow: 0 6px 12px rgba(0, 0, 0, 0.3);
                background-color: #e2e6ea;
                transform: scale(1.05);
            }}
            .status-running {{ background-color: #d4edda; color: #155724; }}
            .status-trade-not-taken {{ background-color: #cce5ff; color: #004085; }}
            .status-max-rentry-reached {{ background-color: #f8d7da; color: #721c24; }}
            .status-max-loss {{ background-color: #ffffff; color: #721c24; animation: blink 1.5s infinite; }} 
            .pnl-positive {{ color: green; }}
            .pnl-negative {{ color: red; }}
            @keyframes blink {{
                0%, 100% {{ opacity: 1; }}
                50% {{ opacity: 0; }}
            }}
            @media (max-width: 768px) {{
                .system {{
                    width: calc(100% - 40px);
                }}
            }}
        </style>
        <script>
            function showDropdown(superName, systemName, indexName) {{
                window.open('/csv/' + indexName + '/' + superName + '/' + systemName, '_blank');
            }}
        </script>
    </head>
    <body>
        <h1>Trading Status Dashboard</h1>
        <div class='timestamp'>Last Updated: {current_time}</div>
        <div class='container'>
    """

    for system_name, super_names, status_values, total_pnls, last_modified_times, folder_name in systems_data:
        html += f"""
        <div class='system'>
            <h2>{system_name}</h2>
            <table class='table'>
                <tr>
                    <th class='th'>INDEX</th>  <!-- Add Index Header -->
                    <th class='th'>Super Name</th>
                    <th class='th'>Status</th>
                    <th class='th'>Total PNL</th>
                    <th class='th'>Last Modified Time</th>
                    <th class='th'>Starting Time</th>
                </tr>
        """
        for super_name in super_names:
            status = status_values[super_names.index(super_name)]
            pnl_value = total_pnls.get(super_name, 'N/A')
            if isinstance(pnl_value, (int, float)):
                pnl_value = round(pnl_value, 2)
            pnl_class = 'pnl-positive' if isinstance(pnl_value, (int, float)) and pnl_value > 0 else 'pnl-negative' if isinstance(pnl_value, (int, float)) and pnl_value < 0 else ''
            status_class = get_status_class(status)
            last_modified_time = last_modified_times.get(super_name, 'N/A')
            normalized_super_name = super_name.strip().upper()
            starting_time = STARTING_TIMES.get(normalized_super_name, "N/A")
            html += f"""
                <tr onclick="showDropdown('{super_name}', '{system_name}', '{folder_name}')">
                    <td class='td'>{folder_name}</td>  <!-- Display Folder Name -->
                    <td class='td'>{super_name}</td>
                    <td class='td {status_class}'>{status}</td>
                    <td class='td {pnl_class}'>{pnl_value}</td>
                    <td class='td'>{last_modified_time}</td>
                    <td class='td'>{starting_time}</td>
                </tr>
            """
        html += "</table></div>"  # Close table and system div

    html += """
        </div>  <!-- Close container -->
    </body>
    </html>
    """
    return render_template_string(html)

def get_status_class(status):
    if status == 'RUNNING':
        return 'status-running'
    elif status == 'MAX LOSS':
        return 'status-max-loss'
    elif status == 'MAX RENTRY REACHED':
        return 'status-max-rentry-reached'
    elif status == 'TRADE NOT TAKEN':
        return 'status-trade-not-taken'
    return ''

if __name__ == '__main__':
    app.run(port=5002, debug=True)
