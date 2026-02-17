import streamlit as st
import json
import pandas as pd
from datetime import datetime
import traceback

# Password protection
def check_password():
    """Returns `True` if the user had the correct password."""
    
    def password_entered():
        """Checks whether a password entered by the user is correct."""
        if st.session_state["password"] == st.secrets.get("auth_password", "defaultpass"):
            st.session_state["password_correct"] = True
            del st.session_state["password"]  # Don't store password
        else:
            st.session_state["password_correct"] = False

    if "password_correct" not in st.session_state:
        # First run, show input for password
        st.text_input(
            "🔒 Enter Password", 
            type="password", 
            on_change=password_entered, 
            key="password"
        )
        st.warning("Please enter your password to access this app")
        return False
    elif not st.session_state["password_correct"]:
        # Password incorrect, show input + error
        st.text_input(
            "🔒 Enter Password", 
            type="password", 
            on_change=password_entered, 
            key="password"
        )
        st.error("😕 Password incorrect")
        return False
    else:
        # Password correct
        return True

# Main app logic
def main():
    st.set_page_config(
        page_title="Maxun Web Scraper",
        page_icon="🔥",
        layout="wide",
        initial_sidebar_state="expanded"
    )

    st.title("🔥 Maxun - Turn Any Website into a Structured API")
    st.markdown("*Powered by AI-driven web extraction with automatic retry and validation*")
    
    # Initialize session state
    if 'extraction_history' not in st.session_state:
        st.session_state.extraction_history = []

    # Sidebar Configuration
    with st.sidebar:
        st.header("⚙️ Configuration")
        
        st.subheader("LLM Settings")
        provider = st.selectbox(
            "Provider",
            ["openai", "xai"],
            help="Choose your LLM provider"
        )
        
        # Default models per provider
        if provider == "openai":
            default_model = "gpt-4o-mini"
            api_key_name = "openai_api_key"
        else:  # xai
            default_model = "grok-beta"
            api_key_name = "xai_api_key"
        
        model = st.text_input(
            "Model", 
            value=default_model,
            help="Specify the model name"
        )
        
        # Get API key from secrets
        api_key = st.secrets.get(api_key_name, "")
        
        if api_key:
            st.success(f"✅ {provider.upper()} API key loaded from secrets")
        else:
            st.error(f"❌ {api_key_name} not found in secrets!")
            api_key = st.text_input(
                f"{provider.capitalize()} API Key",
                type="password",
                help=f"Enter your {provider} API key manually"
            )
        
        st.divider()
        
        st.subheader("Extraction Settings")
        max_retries = st.slider("Max Retries", 1, 10, 3, help="Number of retry attempts")
        temperature = st.slider("Temperature", 0.0, 1.0, 0.1, 0.1, help="LLM creativity")
        headless = st.checkbox("Headless Mode", value=True, help="Run browser in background")
        
        st.divider()
        
        # Show which keys are available
        with st.expander("🔑 Available API Keys"):
            if st.secrets.get("openai_api_key"):
                st.text("✅ OpenAI API key configured")
            if st.secrets.get("xai_api_key"):
                st.text("✅ xAI API key configured")
            if st.secrets.get("auth_password"):
                st.text("✅ Auth password configured")
        
        # Logout button
        if st.button("🚪 Logout", use_container_width=True):
            st.session_state.password_correct = False
            st.rerun()

    # Main Content Area
    tab1, tab2, tab3 = st.tabs(["🎯 Extract Data", "📊 History", "📖 Examples"])
    
    with tab1:
        col1, col2 = st.columns([1, 1])
        
        with col1:
            st.subheader("🌐 Target Configuration")
            url = st.text_input(
                "Website URL",
                placeholder="https://example.com/products",
                help="Enter the URL you want to scrape"
            )
            
            instruction = st.text_area(
                "Extraction Instructions",
                height=200,
                placeholder="Example: Extract all products with their title, price, image URL, and product link. Also indicate if the item is on sale.",
                help="Describe what data you want to extract in natural language"
            )
        
        with col2:
            st.subheader("📋 Schema (Optional)")
            schema_input = st.text_area(
                "JSON Schema",
                height=200,
                placeholder='''{
  "type": "array",
  "items": {
    "type": "object",
    "properties": {
      "title": {"type": "string"},
      "price": {"type": "number"},
      "url": {"type": "string"},
      "image": {"type": "string"}
    },
    "required": ["title", "price"]
  }
}''',
                help="Leave empty for auto-inference, or provide a JSON schema"
            )
        
        if st.button("🚀 Extract Data", type="primary", use_container_width=True):
            if not url or not instruction:
                st.error("⚠️ Please provide both URL and extraction instructions")
            elif not api_key:
                st.error(f"⚠️ Please provide {provider.capitalize()} API key in Streamlit secrets")
            else:
                extract_data(url, instruction, schema_input, provider, model, api_key, 
                           max_retries, temperature, headless)
    
    with tab2:
        show_history()
    
    with tab3:
        show_examples()

def extract_data(url, instruction, schema_input, provider, model, api_key, 
                max_retries, temperature, headless):
    """Perform the actual data extraction"""
    
    progress_bar = st.progress(0)
    status_text = st.empty()
    
    try:
        status_text.text("🔧 Initializing Maxun client...")
        progress_bar.progress(10)
        
        # Import here to avoid issues if not installed
        try:
            from maxun import MaxunClient
        except ImportError:
            st.error("❌ Maxun is not installed. Please add 'maxun' to requirements.txt")
            return
        
        # Initialize client
        # Note: xAI uses OpenAI-compatible API, so we use "openai" as provider
        actual_provider = "openai" if provider == "xai" else provider
        
        # For xAI, we need to set a custom base URL
        if provider == "xai":
            import os
            os.environ["OPENAI_BASE_URL"] = "https://api.x.ai/v1"
        
        client = MaxunClient(
            llm_provider=actual_provider,
            llm_model=model,
            llm_api_key=api_key,
            temperature=temperature,
        )
        
        progress_bar.progress(25)
        status_text.text("🌐 Launching browser...")
        
        # Parse schema if provided
        schema = None
        if schema_input.strip():
            try:
                schema = json.loads(schema_input)
            except json.JSONDecodeError as e:
                st.error(f"❌ Invalid JSON schema: {str(e)}")
                return
        
        progress_bar.progress(40)
        status_text.text("🔍 Extracting data from webpage...")
        
        # Perform extraction
        result = client.extract(
            url=url,
            instruction=instruction,
            schema=schema,
            max_retries=max_retries,
            headless=headless
        )
        
        progress_bar.progress(90)
        status_text.text("✅ Processing results...")
        
        # Store in history
        extraction_record = {
            'timestamp': datetime.now().strftime("%Y-%m-%d %H:%M:%S"),
            'url': url,
            'instruction': instruction,
            'provider': provider,
            'model': model,
            'result': result,
            'count': len(result) if isinstance(result, list) else 1
        }
        st.session_state.extraction_history.insert(0, extraction_record)
        
        # Keep only last 10 extractions
        st.session_state.extraction_history = st.session_state.extraction_history[:10]
        
        progress_bar.progress(100)
        status_text.text("✨ Extraction complete!")
        
        # Display results
        display_results(result, url)
        
    except Exception as e:
        st.error(f"❌ Extraction failed: {str(e)}")
        with st.expander("🔍 View full error traceback"):
            st.code(traceback.format_exc())
    finally:
        progress_bar.empty()
        status_text.empty()

def display_results(result, url):
    """Display extraction results with multiple views"""
    
    st.success(f"✅ Successfully extracted data from {url}")
    
    # Metrics
    if isinstance(result, list):
        col1, col2, col3 = st.columns(3)
        with col1:
            st.metric("Items Extracted", len(result))
        with col2:
            if result and isinstance(result[0], dict):
                st.metric("Fields per Item", len(result[0].keys()))
        with col3:
            st.metric("Data Type", "List of Objects")
    
    st.divider()
    
    # Display tabs
    result_tab1, result_tab2, result_tab3 = st.tabs(["📊 Table View", "🗂️ JSON View", "💾 Download"])
    
    with result_tab1:
        if isinstance(result, list) and result and isinstance(result[0], dict):
            df = pd.DataFrame(result)
            st.dataframe(df, use_container_width=True, height=400)
        else:
            st.json(result)
    
    with result_tab2:
        st.json(result, expanded=True)
    
    with result_tab3:
        col1, col2 = st.columns(2)
        
        with col1:
            json_str = json.dumps(result, indent=2, ensure_ascii=False)
            st.download_button(
                label="📥 Download JSON",
                data=json_str,
                file_name=f"extraction_{datetime.now().strftime('%Y%m%d_%H%M%S')}.json",
                mime="application/json",
                use_container_width=True
            )
        
        with col2:
            if isinstance(result, list) and result and isinstance(result[0], dict):
                df = pd.DataFrame(result)
                csv = df.to_csv(index=False)
                st.download_button(
                    label="📥 Download CSV",
                    data=csv,
                    file_name=f"extraction_{datetime.now().strftime('%Y%m%d_%H%M%S')}.csv",
                    mime="text/csv",
                    use_container_width=True
                )

def show_history():
    """Display extraction history"""
    st.subheader("📊 Extraction History")
    
    if not st.session_state.extraction_history:
        st.info("No extraction history yet. Start by extracting some data!")
        return
    
    for idx, record in enumerate(st.session_state.extraction_history):
        with st.expander(
            f"🕐 {record['timestamp']} - {record['url'][:50]}... ({record['count']} items)",
            expanded=(idx == 0)
        ):
            col1, col2 = st.columns([2, 1])
            
            with col1:
                st.write("**URL:**", record['url'])
                st.write("**Instruction:**", record['instruction'])
                st.write("**Model:**", f"{record['provider']} - {record['model']}")
            
            with col2:
                st.metric("Items", record['count'])
            
            st.json(record['result'])

def show_examples():
    """Show example use cases"""
    st.subheader("📖 Example Use Cases")
    
    examples = [
        {
            "title": "🛍️ E-commerce Product Scraping",
            "url": "https://www.amazon.com/s?k=laptop",
            "instruction": "Extract all laptops with their title, current price, original price (if on sale), rating, number of reviews, and product URL"
        },
        {
            "title": "💼 Job Listings",
            "url": "https://www.indeed.com/jobs?q=software+engineer",
            "instruction": "Extract job listings with title, company name, location, salary range (if available), job type (full-time/part-time), and apply URL"
        },
        {
            "title": "🏠 Real Estate Listings",
            "url": "https://www.zillow.com/homes/for_sale/",
            "instruction": "Extract property listings with address, price, bedrooms, bathrooms, square footage, and listing URL"
        },
        {
            "title": "📰 News Articles",
            "url": "https://news.ycombinator.com/",
            "instruction": "Extract top stories with title, points, number of comments, author, and article link"
        },
        {
            "title": "📚 Research Papers",
            "url": "https://arxiv.org/list/cs.AI/recent",
            "instruction": "Extract recent AI papers with title, authors, abstract, publication date, and PDF link"
        }
    ]
    
    for example in examples:
        with st.expander(example["title"]):
            st.code(f"URL: {example['url']}", language=None)
            st.code(f"Instruction: {example['instruction']}", language=None)
            
            if st.button(f"Use this example", key=example['title']):
                st.info("Copy the URL and instruction to the Extract Data tab!")

# Run the app
if __name__ == "__main__":
    if check_password():
        main()
