# Admin Interface

## Overview

The Arrigoo CDP Admin Interface is a web-based control panel that allows administrators and analysts to explore and manage customer data, define audiences, track events, and automate actions. It is designed to provide both high-level insights and fine-grained control over how data is collected, organized, and activated.

The interface is divided into several sections accessible via the left-hand sidebar, including Dashboards, Profiles, Segments, Events, Properties, Actions, and Integrations. Each section plays a role in helping you understand your users and use your data more effectively.

This documentation provides an overview of each part of the admin interface, including descriptions, typical use cases, and visuals to guide you through its functionality.

## Sidebar Navigation

The left-hand sidebar provides quick access to all major features in the admin interface:

- **Dashboard** – Overview of key metrics and system activity
- **Profiles** – Search, view, and manage individual user profiles
- **Segments** – Build and maintain audience segments based on behavior or attributes
- **Events** – Explore and analyze captured user activity
- **Properties** – Define and manage custom attributes for profiles and events
- **Actions** – Set up automations such as audience syncing or triggered workflows
- **Integrations** – Connect to third-party systems like CRMs, email platforms, or analytics tools

## Dashboard

The **Dashboard** provides an overview of the activity within your CDP. It is the landing view after logging into the admin interface and offers a snapshot of key metrics across your profiles, events, and segments.

### CDP Status Panels

The dashboard displays three main charts:

#### New Profiles Per Day
A line chart showing the number of new profiles created each day.

- **Y-axis**: Number of new profiles
- **X-axis**: Dates
- Useful for monitoring user acquisition trends and detecting spikes or drops in activity

#### Number of Profiles in Segments
A bar chart comparing the size of key segments in your CDP.

- Each bar represents a user segment
- Helps you evaluate audience distribution and segment growth

#### Events Per Day
A stacked bar chart visualizing the volume and variety of events logged daily.

- **Y-axis**: Number of events
- **X-axis**: Dates
- Each color in the stack corresponds to a different event type
- Useful for identifying behavior trends and validating data capture

## Profiles

The **Profiles** section gives you access to all the user profiles stored in the CDP. It is divided into two main views: a high-level dashboard overview and a detailed profile listing.

### Profile Dashboard

The dashboard provides a quick summary of the total number of profiles and how many of them have been identified (i.e., associated with identifiers such as email addresses).

#### Key metrics displayed:
- **Total Profiles / Identified:** Displays the total number of user profiles and how many are identified.
- **Identification Rate:** Shows the percentage of profiles that include at least one identifier.
- **Look up profile:** A utility to search for specific profiles by identifier (e.g., Profile ID or email).

Use this dashboard to quickly assess CDP coverage and effectiveness in identifying users across your data sources.

### Profile Listing

This view presents a paginated table of individual user profiles.

#### Table columns include:
- **Profile ID:** The unique identifier for the profile within the CDP.
- **Segments:** Shows which segments the profile belongs to (e.g., `has-subscribed-to-newsletter`, `binge_reader`).
- **Created / Updated:** Timestamps for when the profile was first created and last updated.
- **Identifiers:** Known identifiers associated with the profile (e.g., email address).
- **Actions:** A quick access button to view or inspect the full profile details.

You can filter profiles using the dropdown in the upper-right corner, which allows toggling between **All** and **Identified** profiles.

This interface is useful for browsing or manually inspecting profiles to verify segmentation logic, check recent updates, or debug identity resolution.

### Profiles – Single Profile View

When you click on a profile in the listing, you are taken to a detailed profile view where all available data about that individual profile is shown.

#### Tabs

The profile details are divided into the following tabs:

- **Properties**: Displays key-value pairs of profile properties, such as email, name, and custom properties like "Total pageviews", "Newsletter ID", or "Interests".
- **Segments**: Shows which segments the profile belongs to.
- **Events**: Lists events tied to the profile.
- **Raw**: Shows the raw profile data in JSON format.

#### Actions

- **Back**: Returns you to the profile list.
- **Delete profile**: Permanently deletes the profile and all associated data.

> ⚠️ Deleting a profile is irreversible and should be used with caution.

This view provides a complete snapshot of the profile's identity, behavior, and segmentation — crucial for support, debugging, or manual inspection tasks.

## Segments

The **Segments** section in Arrigoo CDP lets you define and manage dynamic user segments based on user behavior and profile data.

### Segments Overview

The segments overview table displays the following columns:

| Column       | Description                                                                 |
|--------------|-----------------------------------------------------------------------------|
| **Title**    | The name of the segment (e.g., *Top reader*, *Binge reader*)               |
| **Description** | A human-readable explanation of the segment rule logic                    |
| **Webhooks** | Indicates if a webhook is triggered when the segment is updated      |
| **Members**  | Number of profiles currently in the segment                                |
| **Actions**  | View or Edit the segment definition                                    |


### Actions

- **Add segment**  
  Use the green button in the top right to create a new segment based on filters.

- **View**  
  Opens a detailed view of the segment and its members.

- **Edit**  
  Opens the segment editor to change its logic or metadata.

## Segment Detail Page

The Segment Detail page provides detailed information and visual insights about a selected profile segment.

### Overview

- **Page title:**  
  `Segment: <segment name>` 

- **Charts:**
  - **Line chart (left):**  
    Shows the number of members added and removed from the segment over time.
  - **Bar chart (middle):**  
    Displays the total number of memebers in the segment and members who have left the segment.
  - **Stat box (right):**  
    Highlights what portion of total profiles this segment represents.  


### Description Panel

- **Content:** A plain-text description of the segment rule.  

### Controls

- `Edit` – Opens the segment rule editor.
- `Back` – Returns to the segment list view.

## Events

This page displays and manages all defined event types in the system.

### Table Columns

| Column     | Description                                                                 |
|------------|-----------------------------------------------------------------------------|
| Event      | Name of the event type            |
| Topics     | Whether the event is allowed to have associated topics                      |
| String     | Rules for string properties: Allowed, Required, or Disallowed               |
| Int        | Rules for integer properties: Allowed, Required, or Disallowed              |
| Triggers   | Indicates what triggers are activated by this type of events                  |
| Actions    | Options to edit or delete the event type                                    |


### Button

- **Add event type**: Opens a form to create a new event type.

### Notes

- All editing and deletion is handled inline via the **Actions** column.
- This overview helps maintain control over data hygiene and enforce structure in event tracking.


