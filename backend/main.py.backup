
# Import FastAPI tools for the API and error handling.
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware

# Validate incoming campaign data.
from pydantic import BaseModel, Field, model_validator

# Import SQLAlchemy tools for SQLite database operations.
from sqlalchemy import create_engine, Column, Integer, String, Float, Text
from sqlalchemy.orm import declarative_base, sessionmaker

# Import JSON for saving assumptions and forecasts.
import json


# Create the FastAPI application.
app = FastAPI(title="Qarba Ad Simulator API", version="0.3.0")

# Allow the Next.js frontend to communicate with the backend.
app.add_middleware(
    CORSMiddleware,
    allow_origins=[
        "http://localhost:3000",
        "http://127.0.0.1:3000",
    ],
    allow_credentials=False,
    allow_methods=["GET", "POST"],
    allow_headers=["*"],
)


# Connect to the existing SQLite database.
DATABASE_URL = "sqlite:///./qarba_campaigns.db"

engine = create_engine(
    DATABASE_URL,
    connect_args={"check_same_thread": False},
)

# Create a reusable database session factory.
SessionLocal = sessionmaker(
    autocommit=False,
    autoflush=False,
    bind=engine,
)

# Define the base class for database tables.
Base = declarative_base()


# Provide a short estimate notice for the application.
ESTIMATE_NOTE = (
    "All figures are estimates. Actual results may be better "
    "or worse depending on campaign performance."
)


# Keep track of the current benchmark status.
BENCHMARK_STATUS = {
    "status": "provisional",
    "verified_benchmark_dataset_loaded": False,
    "meta_marketing_api_connected": False,
}


# Define the existing campaign table.
class Campaign(Base):
    __tablename__ = "campaigns"

    # Unique ID for each saved campaign.
    id = Column(Integer, primary_key=True, index=True)

    # Campaign name.
    name = Column(String(150), nullable=False)

    # Total advertising budget in naira.
    budget = Column(Float, nullable=False)

    # Campaign duration.
    duration_days = Column(Integer, nullable=False)

    # Target audience.
    audience = Column(String(30), nullable=False)

    # Save assumptions and forecasts as JSON strings.
    assumptions_json = Column(Text, nullable=False)
    forecast_json = Column(Text, nullable=False)


# Create the table if it does not already exist.
# Existing campaign records and database columns are preserved.
Base.metadata.create_all(bind=engine)


# Define the assumptions for one forecast scenario.
class ScenarioAssumptions(BaseModel):
    # Cost per 1,000 impressions, in naira.
    cpm: float = Field(gt=0)

    # Link click-through rate, as a percentage.
    ctr: float = Field(ge=0, le=100)

    # Estimated percentage of clicks that generate enquiries.
    enquiry_rate: float = Field(ge=0, le=100)


# Define the inputs for a campaign forecast.
class CampaignInput(BaseModel):
    # Total campaign budget.
    budget: float = Field(gt=0, le=100_000_000)

    # Campaign duration in days.
    duration_days: int = Field(ge=1, le=365)

    # Intended audience.
    audience: str

    # Geographic targeting choice.
    target_location: str = "nigeria"

    # Optional country names for selected-country targeting.
    selected_countries: str = ""

    # Assumptions for the three forecast scenarios.
    conservative: ScenarioAssumptions
    expected: ScenarioAssumptions
    optimistic: ScenarioAssumptions

    # Validate the audience and geographic targeting values.
    @model_validator(mode="after")
    def validate_campaign(self):
        if self.audience not in {"buyers", "tenants", "both"}:
            raise ValueError(
                "Audience must be buyers, tenants, or both."
            )

        if self.target_location not in {
            "nigeria",
            "selected_countries",
            "worldwide",
        }:
            raise ValueError(
                "Target location must be nigeria, "
                "selected_countries, or worldwide."
            )

        if (
            self.target_location == "selected_countries"
            and not self.selected_countries.strip()
        ):
            raise ValueError(
                "Enter at least one country for selected-country targeting."
            )

        return self


# Add a name when saving a campaign.
class SaveCampaignInput(CampaignInput):
    name: str = Field(min_length=1, max_length=150)


# Return a readable geographic targeting label.
def location_label(target_location, selected_countries=""):
    if target_location == "nigeria":
        return "Nigeria"

    if target_location == "worldwide":
        return "Worldwide"

    countries = selected_countries.strip()
    return f"Selected countries: {countries}" if countries else "Selected countries"


# Calculate results for one scenario.
def calculate_forecast(budget, duration_days, assumptions):
    # Estimate the number of impressions.
    impressions = (budget / assumptions.cpm) * 1000

    # Estimate link clicks using the click-through rate.
    clicks = impressions * (assumptions.ctr / 100)

    # Estimate enquiries using the assumed enquiry rate.
    enquiries = clicks * (assumptions.enquiry_rate / 100)

    # Calculate costs and return the forecast metrics.
    return {
        "impressions": round(impressions),
        "estimated_link_clicks": round(clicks),
        "estimated_enquiries": round(enquiries, 2),
        "daily_budget": round(budget / duration_days, 2),
        "cost_per_click": round(budget / clicks, 2)
        if clicks > 0 else None,
        "cost_per_enquiry": round(budget / enquiries, 2)
        if enquiries > 0 else None,
    }


# Calculate all three scenarios for a campaign.
def build_forecast(campaign):
    # Group the supplied assumptions.
    scenario_inputs = {
        "conservative": campaign.conservative,
        "expected": campaign.expected,
        "optimistic": campaign.optimistic,
    }

    # Calculate each scenario using the same formulas.
    results = {
        name: calculate_forecast(
            campaign.budget,
            campaign.duration_days,
            assumptions,
        )
        for name, assumptions in scenario_inputs.items()
    }

    # Record the exact assumptions used.
    assumptions_used = {
        name: assumptions.model_dump()
        for name, assumptions in scenario_inputs.items()
    }

    # Return the forecast and its selected geographic target.
    return {
        "currency": "NGN",
        "audience": campaign.audience,
        "target_location": campaign.target_location,
        "selected_countries": campaign.selected_countries,
        "location_label": location_label(
            campaign.target_location,
            campaign.selected_countries,
        ),
        "budget": campaign.budget,
        "duration_days": campaign.duration_days,
        "results_are_estimates": True,
        "estimate_note": ESTIMATE_NOTE,
        "model_version": "0.3.0",
        "benchmark_status": BENCHMARK_STATUS,
        "assumptions_used": assumptions_used,
        "scenarios": results,
    }


# Check whether the backend is running.
@app.get("/")
def home():
    return {
        "message": "Qarba Ad Simulator API is running",
        "version": "0.3.0",
    }


# Return the current benchmark connection status.
@app.get("/benchmark-status")
def benchmark_status():
    return {
        **BENCHMARK_STATUS,
        "model_version": "0.3.0",
        "message": (
            "Location-based assumptions are editable planning inputs. "
            "No verified location-specific benchmark dataset is connected."
        ),
    }


# Calculate a forecast without saving it.
@app.post("/forecast")
def forecast(campaign: CampaignInput):
    return build_forecast(campaign)


# Save a campaign and its forecast in the existing SQLite database.
@app.post("/campaigns")
def save_campaign(campaign: SaveCampaignInput):
    db = SessionLocal()

    try:
        # Calculate the forecast from the submitted settings.
        forecast_result = build_forecast(campaign)

        # Store location and country selection alongside the assumptions.
        assumptions = {
            "target_location": campaign.target_location,
            "selected_countries": campaign.selected_countries,
            "conservative": campaign.conservative.model_dump(),
            "expected": campaign.expected.model_dump(),
            "optimistic": campaign.optimistic.model_dump(),
        }

        # Save using the existing database columns.
        saved = Campaign(
            name=campaign.name.strip(),
            budget=campaign.budget,
            duration_days=campaign.duration_days,
            audience=campaign.audience,
            assumptions_json=json.dumps(assumptions),
            forecast_json=json.dumps(forecast_result),
        )

        db.add(saved)
        db.commit()
        db.refresh(saved)

        # Return the saved campaign.
        return {
            "id": saved.id,
            "name": saved.name,
            "budget": saved.budget,
            "duration_days": saved.duration_days,
            "audience": saved.audience,
            "target_location": campaign.target_location,
            "selected_countries": campaign.selected_countries,
            "assumptions": assumptions,
            "forecast": forecast_result,
        }

    except Exception:
        # Undo unfinished database changes if saving fails.
        db.rollback()
        raise

    finally:
        # Always close the database session.
        db.close()


# Retrieve saved campaigns, newest first.
@app.get("/campaigns")
def list_campaigns():
    db = SessionLocal()

    try:
        campaigns = (
            db.query(Campaign)
            .order_by(Campaign.id.desc())
            .all()
        )

        # Support older records that do not contain location data.
        output = []

        for item in campaigns:
            assumptions = json.loads(item.assumptions_json)
            saved_forecast = json.loads(item.forecast_json)

            target_location = assumptions.get(
                "target_location",
                saved_forecast.get("target_location", "nigeria"),
            )

            selected_countries = assumptions.get(
                "selected_countries",
                saved_forecast.get("selected_countries", ""),
            )

            output.append({
                "id": item.id,
                "name": item.name,
                "budget": item.budget,
                "duration_days": item.duration_days,
                "audience": item.audience,
                "target_location": target_location,
                "selected_countries": selected_countries,
                "assumptions": assumptions,
                "forecast": saved_forecast,
            })

        return output

    finally:
        db.close()


# Retrieve one saved campaign by its ID.
@app.get("/campaigns/{campaign_id}")
def get_campaign(campaign_id: int):
    db = SessionLocal()

    try:
        item = db.query(Campaign).filter(
            Campaign.id == campaign_id
        ).first()

        if item is None:
            raise HTTPException(
                status_code=404,
                detail="Campaign not found",
            )

        assumptions = json.loads(item.assumptions_json)
        saved_forecast = json.loads(item.forecast_json)

        # Fall back to Nigeria for campaigns saved before location targeting.
        target_location = assumptions.get(
            "target_location",
            saved_forecast.get("target_location", "nigeria"),
        )

        selected_countries = assumptions.get(
            "selected_countries",
            saved_forecast.get("selected_countries", ""),
        )

        return {
            "id": item.id,
            "name": item.name,
            "budget": item.budget,
            "duration_days": item.duration_days,
            "audience": item.audience,
            "target_location": target_location,
            "selected_countries": selected_countries,
            "assumptions": assumptions,
            "forecast": saved_forecast,
        }

    finally:
        db.close()
