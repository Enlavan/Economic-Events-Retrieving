# Economic-Events-Retrieving
Efficient Data Retrieval from the EODHD Economic Events Data API

In numerous scenarios, developers are tasked with retrieving large datasets from APIs that impose limits on the number of records returned per request. For example, the Economic Events Data API caps the number of records at 1000 per call. This limitation necessitates a strategic approach to data fetching to ensure comprehensive data retrieval without repeatedly hitting the limit, which could result in incomplete data and wasted resources.
Understanding API Limitations
The Economic Events Data API employs several parameters to effectively manage and tailor data delivery:
    • API Token: Required for authentication. This string key is provided after subscribing to the services.
    • Limit: An integer that restricts the number of records per response, with a possible maximum of 1000. The default value is 50, allowing for control over the data volume per request.
    • Offset: An integer that allows skipping a specified number of records, up to 1000, essential for implementing pagination in data fetching. The default is 0.
    • From and To: Strings that specify the time range for the requested data in the format ‘YYYY-MM-DD’ (e.g., from=2020-01-05 to=2020-02-10). These parameters are crucial for managing the data volume per request.
    • Country: An optional string specifying the country code in ISO 3166 format, consisting of 2 symbols.
    • Comparison: An optional string that allows specifying comparison metrics such as month-over-month (mom), quarter-over-quarter (qoq), or year-over-year (yoy).
Despite these parameters aiding in efficient data request management, a response containing exactly 1000 records often indicates that not all available data has been fetched, suggesting the possibility of more data available beyond the set limit. This situation necessitates additional requests to ensure complete data retrieval, especially when precise data thresholds are reached.
Proposed Algorithm for Data Retrieval
To counter these limitations, the proposed algorithm optimizes the number of API calls while ensuring that all available data within a specified period is retrieved efficiently. Here’s an overview of the implemented algorithm:
    1. Initialization:
        ◦ Define an initial search window (current_delta) and dynamically adjust it based on the returned data.
        ◦ step_up and step_down control the fine-tuning of the time interval based on API response saturation (1000 records).
    2. Data Fetching Loop:
        ◦ For each iteration, determine the end date of the subinterval using the current start date and current_delta.
        ◦ Fetch data for the defined subinterval.
        ◦ If the fetched data is less than the limit, it is safe to assume all data for that period has been retrieved, and the start date is moved forward. If the limit is reached, reduce the subinterval using step_down to avoid hitting the limit repeatedly and ensure all data for the period is captured.
    3. Interval Adjustment:
        ◦ Upon hitting the limit, significantly reduce the interval (step_down) to refine the search and avoid missing data between checks.
        ◦ Once a subinterval with fewer than 1000 records is found, increase the interval (step_up) for the next fetch to optimize further data retrieval.
Significance and Advantages of the Algorithm
This approach offers several advantages:
    • Efficiency: Reduces the number of API calls by dynamically adjusting the interval based on the data volume returned in each call.
    • Coverage: Ensures comprehensive data retrieval by adjusting the query parameters in response to API limits.
    • Scalability: Can be adapted to different APIs with similar constraints, making it a versatile solution for many data retrieval tasks.
Considerations:
    • The loop while len(response) >= limit might lead to multiple API calls if the data density is high. This approach does not guarantee that reducing the interval by step_down will return fewer than the limit unless it sufficiently reduces the interval size based on the actual data density.
    • Boundary Cases: If current_delta is reduced to 1 and still returns the limit, the script will continue hitting the API without progressing the date, potentially causing excessive API calls. This needs to be handled more robustly, possibly with a mechanism to break out of the loop if no suitable interval can be found.
    • Date Handling: The script correctly calculates current_end to never exceed end_date, which is crucial for preventing out-of-bound errors.

import pandas as pd
from eodhd import APIClient
import datetime
api = APIClient('PLACE_YOUR_API_KEY_HERE')

def get_data_for_period(api, start_date, end_date, limit=1000, country=None, comparison=None):
    data = []  # List to store data retrieved from the API
    current_start = start_date  # Initialize the starting date for data retrieval
    current_delta = 300  # Initial assumption of a large time interval which may need adjustment
    step_up = 10  # Increment step for expanding the interval when fewer records are returned
    step_down = 50  # Decrement step for reducing the interval when the limit is reached

    while current_start <= end_date:
        # Calculate the ending date of the current interval being queried
        current_end = min(current_start + datetime.timedelta(days=current_delta), end_date)
        # Request data from the API within the defined date range
        response = api.get_economic_events_data(
            date_from=current_start.strftime('%Y-%m-%d'), 
            date_to=current_end.strftime('%Y-%m-%d'), 
            limit=limit, 
            country=country, 
            comparison=comparison
        )
        
        if len(response) < limit:
            # If the response contains fewer records than the limit, it's safe to assume all records for the period have been fetched
            data.extend(response)  # Add the retrieved data to the list
            current_start = current_end + datetime.timedelta(days=1)  # Move the start date to the day after the current end date
            current_delta += step_up  # Increase the interval for the next query to potentially capture more data
        else:
            # If the response contains records at the limit, adjust the query interval to avoid missing data
            while len(response) >= limit:
                current_delta -= step_down  # Decrease the interval to find a range that does not hit the limit
                if current_delta <= 0:
                    current_delta = 1  # Ensure the interval does not become zero or negative
                current_end = current_start + datetime.timedelta(days=current_delta)  # Update the end date based on the new interval
                response = api.get_economic_events_data(
                    date_from=current_start.strftime('%Y-%m-%d'), 
                    date_to=current_end.strftime('%Y-%m-%d'), 
                    limit=limit, 
                    country=country, 
                    comparison=comparison
                )  # Re-query with the adjusted interval

            data.extend(response)  # Once an appropriate interval is found, add the data to the list
            current_start = current_end + datetime.timedelta(days=1)  # Move the start date to the day after the adjusted end date

    return data  # Return the collected data

# Example usage:
start_date = datetime.datetime.strptime('2020-01-05', '%Y-%m-%d')
end_date = datetime.datetime.strptime('2024-07-22', '%Y-%m-%d')
limit = 1000
country = 'US'
economic_events = get_data_for_period(api, start_date, end_date, limit, country)
df = pd.DataFrame(economic_events)
df
This script will output a dataframe containing all records from the Economic Events API Data for the specified time period and for the specified parameters:

Conclusion
Efficiently retrieving data from APIs with strict limits on the number of records per call is crucial for comprehensive data analysis and application performance. The proposed algorithm not only adheres to these limits but also optimizes the data retrieval process, ensuring that all available data is fetched with the fewest possible API calls. This strategy is particularly useful in scenarios where data completeness and retrieval efficiency are critical, such as in financial data analysis.
