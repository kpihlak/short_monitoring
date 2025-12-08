# Splunk Cloud Lab: Hands-On Activity

This lab provides an introduction to Splunk Observability Cloud using its free trial. It guides users through setting up the environment, monitoring sample data, and exploring core features. It also refers to the Splunk Search Tutorial for additional context and exercises.

## Objective
- Get hands-on experience with Splunk Observability Cloud.
- Learn to analyze logs, metrics, and traces in real-time.
- Create dashboards and set alerts.
- Leverage the Splunk Search Tutorial for guided learning.

## Pre-Requisites
- Access to [Splunk Observability Cloud Free Trial](https://www.splunk.com/en_us/download/o11y-cloud-free-trial.html).
- Basic understanding of application monitoring and observability.
- Tutorial data files from [Splunk Documentation](https://docs.splunk.com/Documentation/Splunk/8.0.4/SearchTutorial/Systemrequirements#Download_the_tutorial_data_files).

## Steps

### 1. **Sign Up for Free Trial**
   - Visit the [Splunk Observability Cloud Free Trial](https://www.splunk.com/en_us/download/o11y-cloud-free-trial.html).
   - Register using your email. No credit card is required.

### 2. **Download Tutorial Data**
   - Visit the [Splunk Search Tutorial System Requirements](https://docs.splunk.com/Documentation/Splunk/8.0.4/SearchTutorial/Systemrequirements#Download_the_tutorial_data_files).
   - Download the following files:
     - `tutorialdata.zip` (Do not uncompress the file.)
     - `Prices.csv.zip` (Do not uncompress the file at this time.)
   - Ensure that Safari users disable the "Open safe files after downloading" option in browser preferences.

### 3. **Access Splunk Cloud**
   - Go to the [Splunk website](https://www.splunk.com/).
   - Click **Free Splunk** and download the trial software for your platform.
   - Confirm that you are not a robot and click **Start Trial**.
   - Follow the instructions in the confirmation email to access your Splunk Cloud Trial.
   - Accept the Terms of Service, and Splunk Cloud will open in a browser window.

### 4. **Upload Tutorial Data**
   - Refer to [Part 2: Uploading the tutorial data](https://docs.splunk.com/Documentation/Splunk/8.0.4/SearchTutorial/Aboutthesearchapp) in the Splunk Search Tutorial.
   - In your Splunk Cloud instance, navigate to the **Settings** menu.
   - Select **Add Data** and upload the `tutorialdata.zip` file.
   - Follow the prompts to complete the data upload process.

### 5. **Analyze Logs and Metrics**
   - Navigate to the **Search & Reporting** app.
   - Refer to [Part 3: Using the Splunk Search App](https://docs.splunk.com/Documentation/Splunk/8.0.4/SearchTutorial/Aboutthesearchapp) for detailed examples.
   - Use the uploaded data to perform searches and queries.
   - Explore logs from web access, sales, and pricing data.

### 6. **Create a Dashboard**
   - Go to the **Dashboards** section in Splunk Cloud.
   - Refer to [Part 7: Creating dashboards](https://docs.splunk.com/Documentation/Splunk/8.0.4/SearchTutorial/Aboutthesearchapp) in the Splunk Search Tutorial for additional guidance.
   - Add visualizations like sales trends, pricing changes, and log access patterns.
   - Customize the layout and apply filters for deeper insights.

### 7. **Set Alerts**
   - Navigate to the Alerts section in Splunk Cloud.
   - Create a new alert for specific events, such as high sales volume or errors in log files.
   - Configure thresholds and notification preferences.

### 8. **Review Traces and Synthetic Monitoring**
   - Experiment with Splunk Observability features to view traces and synthetic tests.
   - Monitor application performance and user experience in real-time.

## Expected Outcomes
- Familiarity with Splunk’s Observability Cloud interface.
- Ability to analyze logs, metrics, and traces.
- Skills to create dashboards and configure alerts using tutorial data.
- Enhanced understanding through the Splunk Search Tutorial.

## Additional Resources
- [Splunk Observability Cloud Documentation](https://docs.splunk.com/ObservabilityCloud/)
- [OpenTelemetry Instrumentation Guide](https://opentelemetry.io/docs/)
- [Splunk Search Tutorial](https://docs.splunk.com/Documentation/Splunk/8.0.4/SearchTutorial)
