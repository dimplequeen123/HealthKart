import streamlit as st
import pandas as pd
import plotly.express as px

def load_data(file_path):
    try:
        df = pd.read_csv(file_path)
        # Convert date columns to datetime objects
        for col in df.columns:
            if 'date' in col:
                df[col] = pd.to_datetime(df[col], errors='coerce')
        return df
    except Exception as e:
        st.error(f"Error loading {file_path}: {e}")
        return pd.DataFrame()

# Placeholder for dataframes (will be loaded from user uploads)
df_influencers = pd.DataFrame()
df_posts = pd.DataFrame()
df_tracking = pd.DataFrame()
df_payouts = pd.DataFrame()

# --- Utility Functions ---
def calculate_roas(revenue, payout):
    if payout > 0:
        return (revenue / payout) * 100
    return 0

def get_brand_from_product(product_name):
    if "MuscleBlaze" in product_name:
        return "MuscleBlaze"
    elif "HKVitals" in product_name:
        return "HKVitals"
    elif "Gritzo" in product_name:
        return "Gritzo"
    return "Other"

# --- Streamlit App Layout ---
st.set_page_config(layout="wide", page_title="Influencer ROI Dashboard")

st.title("📈 HealthKart Influencer Campaign ROI Dashboard")

# Sidebar for Data Upload and Global Filters
st.sidebar.header("📂 Data Upload")
uploaded_influencers_file = st.sidebar.file_uploader("Upload influencers.csv", type=["csv"])
uploaded_posts_file = st.sidebar.file_uploader("Upload posts.csv", type=["csv"])
uploaded_tracking_file = st.sidebar.file_uploader("Upload tracking_data.csv", type=["csv"])
uploaded_payouts_file = st.sidebar.file_uploader("Upload payouts.csv", type=["csv"])

if uploaded_influencers_file:
    df_influencers = load_data(uploaded_influencers_file)
if uploaded_posts_file:
    df_posts = load_data(uploaded_posts_file)
if uploaded_tracking_file:
    df_tracking = load_data(uploaded_tracking_file)
    df_tracking['brand'] = df_tracking['product'].apply(get_brand_from_product) # Add brand column
if uploaded_payouts_file:
    df_payouts = load_data(uploaded_payouts_file)

if not (df_influencers.empty or df_posts.empty or df_tracking.empty or df_payouts.empty):
    st.sidebar.success("All data files loaded successfully!")

    st.sidebar.header("⚙️ Global Filters")

    # Date Range Filter
    min_date = df_tracking['date'].min()
    max_date = df_tracking['date'].max()
    date_range = st.sidebar.date_input("Select Date Range", value=(min_date, max_date),
                                        min_value=min_date, max_value=max_date)

    if len(date_range) == 2:
        start_date, end_date = date_range
        df_tracking_filtered = df_tracking[(df_tracking['date'] >= pd.to_datetime(start_date)) &
                                         (df_tracking['date'] <= pd.to_datetime(end_date))]
        df_posts_filtered = df_posts[(df_posts['date'] >= pd.to_datetime(start_date)) &
                                     (df_posts['date'] <= pd.to_datetime(end_date))]
        df_payouts_filtered = df_payouts[(df_payouts['payout_date'] >= pd.to_datetime(start_date)) &
                                         (df_payouts['payout_date'] <= pd.to_datetime(end_date))]
    else:
        st.sidebar.warning("Please select a date range.")
        df_tracking_filtered = df_tracking
        df_posts_filtered = df_posts
        df_payouts_filtered = df_payouts

    # Brand Filter
    all_brands = ['All'] + list(df_tracking['brand'].unique())
    selected_brand = st.sidebar.selectbox("Filter by Brand", all_brands)

    # Product Filter
    products_for_selected_brand = ['All']
    if selected_brand != 'All':
        products_for_selected_brand.extend(df_tracking[df_tracking['brand'] == selected_brand]['product'].unique())
    else:
        products_for_selected_brand.extend(df_tracking['product'].unique())
    selected_product = st.sidebar.selectbox("Filter by Product", products_for_selected_brand)

    # Influencer Category Filter
    all_categories = ['All'] + list(df_influencers['category'].unique())
    selected_category = st.sidebar.selectbox("Filter by Influencer Category", all_categories)

    # Platform Filter
    all_platforms = ['All'] + list(df_posts['platform'].unique())
    selected_platform = st.sidebar.selectbox("Filter by Platform", all_platforms)

    # --- Apply Filters to DataFrames ---
    if selected_brand != 'All':
        df_tracking_filtered = df_tracking_filtered[df_tracking_filtered['brand'] == selected_brand]
    if selected_product != 'All':
        df_tracking_filtered = df_tracking_filtered[df_tracking_filtered['product'] == selected_product]

    # Join influencers data for category/platform filtering
    df_tracking_merged = pd.merge(df_tracking_filtered, df_influencers, on='influencer_id', how='left')
    df_posts_merged = pd.merge(df_posts_filtered, df_influencers, on='influencer_id', how='left')
    df_payouts_merged = pd.merge(df_payouts_filtered, df_influencers, on='influencer_id', how='left')

    if selected_category != 'All':
        df_tracking_merged = df_tracking_merged[df_tracking_merged['category'] == selected_category]
        df_posts_merged = df_posts_merged[df_posts_merged['category'] == selected_category]
        df_payouts_merged = df_payouts_merged[df_payouts_merged['category'] == selected_category]

    if selected_platform != 'All':
        df_tracking_merged = df_tracking_merged[df_tracking_merged['platform_x'] == selected_platform] # platform_x is from tracking_data
        df_posts_merged = df_posts_merged[df_posts_merged['platform_x'] == selected_platform] # platform_x is from posts
        df_payouts_merged = df_payouts_merged[df_payouts_merged['platform_x'] == selected_platform]


    # --- Main Content Area (Tabs) ---
    tab1, tab2, tab3, tab4 = st.tabs(["Campaign Performance", "Influencer Insights", "Post Performance", "Payout Tracking"])

    with tab1:
        st.header("📊 Campaign Performance")

        # Calculate KPIs
        total_revenue_influencer = df_tracking_merged[df_tracking_merged['source'] == 'Influencer Campaign']['revenue'].sum()
        total_payout = df_payouts_merged['total_payout'].sum()
        total_orders_influencer = df_tracking_merged[df_tracking_merged['source'] == 'Influencer Campaign']['orders'].sum()

        overall_roas = calculate_roas(total_revenue_influencer, total_payout)

        # Incremental ROAS
        # Baseline: Revenue NOT attributed to 'Influencer Campaign'
        baseline_revenue = df_tracking_merged[df_tracking_merged['source'] != 'Influencer Campaign']['revenue'].sum()
        incremental_revenue = total_revenue_influencer - baseline_revenue # Simplified for demonstration
        incremental_roas = calculate_roas(incremental_revenue, total_payout) if baseline_revenue < total_revenue_influencer else 0

        col1, col2, col3, col4 = st.columns(4)
        col1.metric("Total Influencer Revenue", f"₹{total_revenue_influencer:,.2f}")
        col2.metric("Total Orders (Influencer)", f"{total_orders_influencer:,}")
        col3.metric("Overall ROAS", f"{overall_roas:,.2f}%")
        col4.metric("Incremental ROAS (Simplified)", f"{incremental_roas:,.2f}%")
        st.info(
            "Incremental ROAS here is calculated as: ((Influencer Revenue - Non-Influencer Revenue Baseline) / Total Payout) * 100. "
            "A negative value implies influencer revenue is less than your baseline non-influencer revenue in the filtered period. "
            "For a true incremental ROAS, robust control groups or advanced attribution models are needed."
        )

        st.subheader("Revenue & Payout by Campaign")
        campaign_summary = df_tracking_merged[df_tracking_merged['source'] == 'Influencer Campaign'].groupby('campaign').agg(
            total_revenue=('revenue', 'sum'),
            total_orders=('orders', 'sum')
        ).reset_index()
        payout_summary = df_payouts_merged.groupby('campaign').agg(
            total_payout=('total_payout', 'sum')
        ).reset_index()

        campaign_performance = pd.merge(campaign_summary, payout_summary, on='campaign', how='outer').fillna(0)
        campaign_performance['ROAS'] = campaign_performance.apply(lambda row: calculate_roas(row['total_revenue'], row['total_payout']), axis=1)

        st.dataframe(campaign_performance.sort_values(by='ROAS', ascending=False))

        fig_campaign_rev = px.bar(campaign_performance, x='campaign', y='total_revenue', title='Total Revenue by Campaign',
                                 labels={'total_revenue': 'Revenue (₹)'}, color='campaign')
        st.plotly_chart(fig_campaign_rev, use_container_width=True)

        fig_campaign_roas = px.bar(campaign_performance, x='campaign', y='ROAS', title='ROAS by Campaign',
                                 labels={'ROAS': 'ROAS (%)'}, color='campaign')
        st.plotly_chart(fig_campaign_roas, use_container_width=True)


    with tab2:
        st.header("👤 Influencer Insights")

        influencer_revenue = df_tracking_merged[df_tracking_merged['source'] == 'Influencer Campaign'].groupby('influencer_id').agg(
            total_revenue=('revenue', 'sum'),
            total_orders=('orders', 'sum')
        ).reset_index()

        influencer_payout = df_payouts_merged.groupby('influencer_id').agg(
            total_payout=('total_payout', 'sum')
        ).reset_index()

        influencer_performance = pd.merge(influencer_revenue, influencer_payout, on='influencer_id', how='outer').fillna(0)
        influencer_performance = pd.merge(influencer_performance, df_influencers[['influencer_id', 'name', 'category', 'follower_count', 'platform']], on='influencer_id', how='left')

        influencer_performance['ROAS'] = influencer_performance.apply(lambda row: calculate_roas(row['total_revenue'], row['total_payout']), axis=1)

        st.subheader("Top Influencers by Revenue")
        st.dataframe(influencer_performance.sort_values(by='total_revenue', ascending=False).head(10))

        st.subheader("Influencers by ROAS")
        st.dataframe(influencer_performance.sort_values(by='ROAS', ascending=False).head(10))

        st.subheader("Poor ROI Influencers")
        # Define a threshold for "poor ROI" - e.g., ROAS < 100% (or significantly lower than average)
        poor_roi_threshold = 100
        poor_roi_influencers = influencer_performance[influencer_performance['ROAS'] < poor_roi_threshold].sort_values(by='ROAS')
        if not poor_roi_influencers.empty:
            st.dataframe(poor_roi_influencers)
        else:
            st.info("No influencers with ROAS below the defined threshold.")

        st.subheader("Best Personas (Category & Gender)")
        persona_performance = influencer_performance.groupby(['category', 'gender']).agg(
            avg_roas=('ROAS', 'mean'),
            total_revenue=('total_revenue', 'sum'),
            influencer_count=('influencer_id', 'count')
        ).reset_index()
        st.dataframe(persona_performance.sort_values(by='avg_roas', ascending=False))

        fig_influencer_roas = px.bar(influencer_performance.sort_values(by='ROAS', ascending=False).head(10),
                                     x='name', y='ROAS', title='Top 10 Influencers by ROAS',
                                     labels={'ROAS': 'ROAS (%)', 'name': 'Influencer Name'})
        st.plotly_chart(fig_influencer_roas, use_container_width=True)

    with tab3:
        st.header("📝 Post Performance")
        df_posts_merged['engagement_rate'] = ((df_posts_merged['likes'] + df_posts_merged['comments']) / df_posts_merged['reach']) * 100
        st.subheader("Top Performing Posts by Engagement Rate")
        st.dataframe(df_posts_merged.sort_values(by='engagement_rate', ascending=False).head(10))

        st.subheader("Posts by Reach and Engagement")
        fig_post_scatter = px.scatter(df_posts_merged, x='reach', y='engagement_rate',
                                      hover_data=['name', 'URL', 'caption'],
                                      title='Post Reach vs. Engagement Rate',
                                      labels={'reach': 'Reach', 'engagement_rate': 'Engagement Rate (%)'})
        st.plotly_chart(fig_post_scatter, use_container_width=True)


    with tab4:
        st.header("💸 Payout Tracking")
        st.subheader("All Payouts")
        st.dataframe(df_payouts_merged)

        st.subheader("Total Payouts by Influencer")
        payout_by_influencer = df_payouts_merged.groupby(['influencer_id', 'name']).agg(
            total_payout=('total_payout', 'sum')
        ).reset_index().sort_values(by='total_payout', ascending=False)
        st.dataframe(payout_by_influencer)

        st.subheader("Payouts by Basis (Post vs. Order)")
        payout_by_basis = df_payouts_merged.groupby('basis').agg(
            total_payout=('total_payout', 'sum')
        ).reset_index()
        fig_payout_basis = px.pie(payout_by_basis, values='total_payout', names='basis',
                                  title='Total Payouts by Basis')
        st.plotly_chart(fig_payout_basis, use_container_width=True)

else:
    st.info("Please upload all four CSV files to see the dashboard. Example data structure is provided in the requirements.")
