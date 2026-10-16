
"use client";

// Import React hooks for dashboard state and campaign history.
import { useCallback, useEffect, useState } from "react";

// Import jsPDF for downloadable campaign reports.
import { jsPDF } from "jspdf";

// FastAPI backend address.
const API_URL = "http://127.0.0.1:8000";

// Supported location options.
type LocationKey = "nigeria" | "selected_countries" | "worldwide";

// Supported forecast scenarios.
type ScenarioKey = "conservative" | "expected" | "optimistic";

// Editable advertising assumptions.
type Scenario = {
  cpm: number;
  ctr: number;
  enquiryRate: number;
};

type ScenarioSet = Record<ScenarioKey, Scenario>;
type LocationProfiles = Record<LocationKey, ScenarioSet>;

// Forecast metrics returned by the backend or calculated locally.
type Forecast = {
  impressions: number;
  estimated_link_clicks: number;
  estimated_enquiries: number;
  daily_budget: number;
  cost_per_click: number | null;
  cost_per_enquiry: number | null;
};

// Forecast response from FastAPI.
type ForecastResponse = {
  currency: string;
  audience: string;
  target_location?: LocationKey;
  selected_countries?: string;
  location_label?: string;
  budget: number;
  duration_days: number;
  results_are_estimates: boolean;
  estimate_note?: string;
  scenarios: Record<string, Forecast>;
};

// Shapes used to restore saved assumptions.
type SavedScenario = {
  cpm?: number;
  ctr?: number;
  enquiryRate?: number;
  enquiry_rate?: number;
};

// A saved campaign returned by the backend.
type SavedCampaign = {
  id: number;
  name: string;
  budget: number;
  duration_days: number;
  audience: string;
  target_location?: LocationKey;
  selected_countries?: string;
  assumptions: Record<string, unknown>;
  forecast: ForecastResponse;
};

// Starting assumptions for Nigeria.
// These are editable planning inputs, not verified Meta benchmarks.
const nigeriaScenarios: ScenarioSet = {
  conservative: { cpm: 3000, ctr: 0.5, enquiryRate: 2 },
  expected: { cpm: 2000, ctr: 1, enquiryRate: 4 },
  optimistic: { cpm: 1500, ctr: 1.5, enquiryRate: 6 },
};

// Illustrative starting assumptions for selected countries.
const selectedCountryScenarios: ScenarioSet = {
  conservative: { cpm: 4000, ctr: 0.4, enquiryRate: 1.5 },
  expected: { cpm: 3000, ctr: 0.8, enquiryRate: 3 },
  optimistic: { cpm: 2000, ctr: 1.2, enquiryRate: 5 },
};

// Illustrative starting assumptions for worldwide targeting.
const worldwideScenarios: ScenarioSet = {
  conservative: { cpm: 6000, ctr: 0.3, enquiryRate: 1 },
  expected: { cpm: 4000, ctr: 0.7, enquiryRate: 2.5 },
  optimistic: { cpm: 2500, ctr: 1.1, enquiryRate: 4 },
};

// Keep separate editable assumptions for each location.
const initialProfiles: LocationProfiles = {
  nigeria: nigeriaScenarios,
  selected_countries: selectedCountryScenarios,
  worldwide: worldwideScenarios,
};

// Readable scenario names.
const scenarioNames: Record<ScenarioKey, string> = {
  conservative: "Conservative",
  expected: "Expected",
  optimistic: "Optimistic",
};

// Readable geographic labels.
function locationLabel(location: LocationKey, countries: string) {
  if (location === "nigeria") return "Nigeria";
  if (location === "worldwide") return "Worldwide";

  return countries.trim()
    ? `Selected countries: ${countries.trim()}`
    : "Selected countries";
}

// Readable audience labels.
function audienceLabel(audience: string) {
  if (audience === "buyers") return "Property buyers";
  if (audience === "tenants") return "Tenants";
  return "Buyers and tenants";
}

// Check whether a value is a valid positive number.
function isPositiveNumber(value: number) {
  return Number.isFinite(value) && value > 0;
}

// Safely restore one scenario from saved campaign data.
function restoreScenario(
  saved: SavedScenario | undefined,
  fallback: Scenario
): Scenario {
  const cpm =
    typeof saved?.cpm === "number" &&
    Number.isFinite(saved.cpm) &&
    saved.cpm > 0
      ? saved.cpm
      : fallback.cpm;

  const ctr =
    typeof saved?.ctr === "number" &&
    Number.isFinite(saved.ctr) &&
    saved.ctr >= 0
      ? saved.ctr
      : fallback.ctr;

  const enquiryRate =
    typeof saved?.enquiryRate === "number" &&
    Number.isFinite(saved.enquiryRate) &&
    saved.enquiryRate >= 0
      ? saved.enquiryRate
      : typeof saved?.enquiry_rate === "number" &&
          Number.isFinite(saved.enquiry_rate) &&
          saved.enquiry_rate >= 0
        ? saved.enquiry_rate
        : fallback.enquiryRate;

  return { cpm, ctr, enquiryRate };
}

// Restore all three scenarios from a saved campaign.
function restoreScenarioSet(
  saved: Record<string, unknown>,
  fallback: ScenarioSet
): ScenarioSet {
  return {
    conservative: restoreScenario(
      saved.conservative as SavedScenario | undefined,
      fallback.conservative
    ),
    expected: restoreScenario(
      saved.expected as SavedScenario | undefined,
      fallback.expected
    ),
    optimistic: restoreScenario(
      saved.optimistic as SavedScenario | undefined,
      fallback.optimistic
    ),
  };
}

// Check whether a saved location value is supported.
function isLocationKey(value: unknown): value is LocationKey {
  return (
    value === "nigeria" ||
    value === "selected_countries" ||
    value === "worldwide"
  );
}

export default function Home() {
  // Campaign settings.
  const [name, setName] = useState("Qarba Campaign");
  const [budget, setBudget] = useState(50000);
  const [days, setDays] = useState(14);
  const [audience, setAudience] = useState("both");

  // Geographic targeting settings.
  const [targetLocation, setTargetLocation] =
    useState<LocationKey>("nigeria");
  const [selectedCountries, setSelectedCountries] = useState("");

  // Store separate assumptions for each geographic option.
  const [profiles, setProfiles] =
    useState<LocationProfiles>(initialProfiles);

  // Get assumptions for the currently selected location.
  const scenarios = profiles[targetLocation];

  // Dashboard activity and response state.
  const [loading, setLoading] = useState(false);
  const [saving, setSaving] = useState(false);
  const [listLoading, setListLoading] = useState(false);
  const [message, setMessage] = useState("");
  const [error, setError] = useState("");
  const [backendResults, setBackendResults] =
    useState<ForecastResponse | null>(null);
  const [savedCampaigns, setSavedCampaigns] =
    useState<SavedCampaign[]>([]);

  // Format amounts in Nigerian naira.
  const money = (value: number | null | undefined) =>
    value == null || !Number.isFinite(value)
      ? "N/A"
      : new Intl.NumberFormat("en-NG", {
          style: "currency",
          currency: "NGN",
          maximumFractionDigits: 2,
        }).format(value);

  // Format numerical metrics for display.
  const number = (value: number | null | undefined) =>
    value == null || !Number.isFinite(value)
      ? "N/A"
      : new Intl.NumberFormat("en-NG", {
          maximumFractionDigits: 2,
        }).format(value);

  // Clear old backend results after a setting changes.
  const invalidateResults = () => {
    setBackendResults(null);
    setMessage("");
    setError("");
  };

  // Prepare the data sent to FastAPI.
  // main.py must define and store the location fields for full support.
  const buildPayload = () => ({
    budget,
    duration_days: days,
    audience,
    target_location: targetLocation,
    selected_countries:
      targetLocation === "selected_countries"
        ? selectedCountries.trim()
        : "",
    conservative: {
      cpm: scenarios.conservative.cpm,
      ctr: scenarios.conservative.ctr,
      enquiry_rate: scenarios.conservative.enquiryRate,
    },
    expected: {
      cpm: scenarios.expected.cpm,
      ctr: scenarios.expected.ctr,
      enquiry_rate: scenarios.expected.enquiryRate,
    },
    optimistic: {
      cpm: scenarios.optimistic.cpm,
      ctr: scenarios.optimistic.ctr,
      enquiry_rate: scenarios.optimistic.enquiryRate,
    },
  });

  // Update one assumption for the selected location.
  const updateScenario = (
    scenarioName: ScenarioKey,
    field: keyof Scenario,
    value: number
  ) => {
    setProfiles((previous) => ({
      ...previous,
      [targetLocation]: {
        ...previous[targetLocation],
        [scenarioName]: {
          ...previous[targetLocation][scenarioName],
          [field]: value,
        },
      },
    }));

    invalidateResults();
  };

  // Validate campaign settings before sending them to the backend.
  const validateCampaign = () => {
    if (!name.trim()) {
      setError("Enter a campaign name.");
      return false;
    }

    if (!isPositiveNumber(budget) || budget > 100000000) {
      setError("Enter a budget between NGN 1 and NGN 100,000,000.");
      return false;
    }

    if (!Number.isInteger(days) || days < 1 || days > 365) {
      setError("Campaign duration must be between 1 and 365 days.");
      return false;
    }

    if (
      targetLocation === "selected_countries" &&
      !selectedCountries.trim()
    ) {
      setError("Enter the countries you want to target.");
      return false;
    }

    for (const [scenarioKey, scenario] of Object.entries(scenarios)) {
      if (!isPositiveNumber(scenario.cpm)) {
        setError(
          `${scenarioNames[scenarioKey as ScenarioKey]} CPM must be greater than zero.`
        );
        return false;
      }

      if (
        !Number.isFinite(scenario.ctr) ||
        scenario.ctr < 0 ||
        scenario.ctr > 100
      ) {
        setError("CTR must be between 0 and 100.");
        return false;
      }

      if (
        !Number.isFinite(scenario.enquiryRate) ||
        scenario.enquiryRate < 0 ||
        scenario.enquiryRate > 100
      ) {
        setError("Enquiry rate must be between 0 and 100.");
        return false;
      }
    }

    return true;
  };

  // Safely parse API responses and surface backend errors.
  const readResponse = async (response: Response) => {
    const data = await response.json().catch(() => ({}));

    if (!response.ok) {
      throw new Error(
        typeof data.detail === "string"
          ? data.detail
          : JSON.stringify(data.detail ?? data)
      );
    }

    return data;
  };

  // Request a forecast from FastAPI.
  const runForecast = async () => {
    setMessage("");
    setError("");

    if (!validateCampaign()) return;

    setLoading(true);

    try {
      const response = await fetch(`${API_URL}/forecast`, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(buildPayload()),
      });

      const data = await readResponse(response);

      if (!data.scenarios) {
        throw new Error("The backend response does not contain forecast scenarios.");
      }

      setBackendResults(data as ForecastResponse);
      setMessage("Forecast calculated successfully.");
    } catch (err) {
      setError(
        err instanceof Error
          ? err.message
          : "Could not calculate the forecast. Check that FastAPI is running."
      );
    } finally {
      setLoading(false);
    }
  };

  // Load saved campaigns from the backend database.
  const loadCampaigns = useCallback(async () => {
    setListLoading(true);

    try {
      const response = await fetch(`${API_URL}/campaigns`);
      const data = await readResponse(response);

      if (!Array.isArray(data)) {
        throw new Error("The backend returned an unexpected campaign list.");
      }

      setSavedCampaigns(data as SavedCampaign[]);
    } catch (err) {
      setError(
        err instanceof Error
          ? err.message
          : "Could not load saved campaigns."
      );
    } finally {
      setListLoading(false);
    }
  }, []);

  // Load campaign history when the dashboard opens.
  useEffect(() => {
    void loadCampaigns();
  }, [loadCampaigns]);

  // Save the campaign and its assumptions through FastAPI.
  const saveCampaign = async () => {
    setMessage("");
    setError("");

    if (!validateCampaign()) return;

    setSaving(true);

    try {
      const response = await fetch(`${API_URL}/campaigns`, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          name: name.trim(),
          ...buildPayload(),
        }),
      });

      const data = await readResponse(response);

      if (!data.forecast) {
        throw new Error(
          "The backend saved a response without a forecast. Check the campaign API."
        );
      }

      setBackendResults(data.forecast as ForecastResponse);
      setMessage(`Campaign "${data.name ?? name.trim()}" saved successfully.`);

      await loadCampaigns();
    } catch (err) {
      setError(
        err instanceof Error ? err.message : "Could not save the campaign."
      );
    } finally {
      setSaving(false);
    }
  };

  // Restore a campaign's settings, assumptions, and saved forecast.
  const viewCampaign = (campaign: SavedCampaign) => {
    const location = isLocationKey(campaign.target_location)
      ? campaign.target_location
      : "nigeria";

    const savedAssumptions = campaign.assumptions ?? {};

    setName(campaign.name ?? "Saved campaign");
    setBudget(campaign.budget ?? 50000);
    setDays(campaign.duration_days ?? 14);
    setAudience(campaign.audience ?? "both");
    setTargetLocation(location);
    setSelectedCountries(campaign.selected_countries ?? "");

    // Restore assumptions for the campaign's target location.
    setProfiles((previous) => ({
      ...previous,
      [location]: restoreScenarioSet(
        savedAssumptions,
        previous[location] ?? initialProfiles[location]
      ),
    }));

    // Use the saved forecast until settings are changed.
    setBackendResults(campaign.forecast ?? null);
    setMessage(`Viewing saved campaign: ${campaign.name}`);
    setError("");

    window.scrollTo({ top: 0, behavior: "smooth" });
  };

  // Calculate local forecasts for immediate dashboard display.
  // Backend results replace these calculations after a successful request.
  const forecasts = (
    Object.entries(scenarios) as [ScenarioKey, Scenario][]
  ).map(([key, scenario]) => {
    const impressions =
      scenario.cpm > 0 ? (budget / scenario.cpm) * 1000 : 0;

    const clicks = impressions * (scenario.ctr / 100);
    const enquiries = clicks * (scenario.enquiryRate / 100);

    const localForecast: Forecast = {
      impressions: Math.round(impressions),
      estimated_link_clicks: Math.round(clicks),
      estimated_enquiries: Math.round(enquiries * 100) / 100,
      daily_budget: budget / days,
      cost_per_click: clicks > 0 ? budget / clicks : null,
      cost_per_enquiry: enquiries > 0 ? budget / enquiries : null,
    };

    return {
      key,
      label: scenarioNames[key],
      forecast: backendResults?.scenarios?.[key] ?? localForecast,
    };
  });

  // Generate and download a PDF report.
  const downloadPdfReport = () => {
    try {
      const doc = new jsPDF({
        orientation: "portrait",
        unit: "mm",
        format: "a4",
      });

      const pageWidth = doc.internal.pageSize.getWidth();
      const pageHeight = doc.internal.pageSize.getHeight();
      const margin = 14;

      // Branded report header.
      doc.setFillColor(30, 64, 175);
      doc.rect(0, 0, pageWidth, 38, "F");

      doc.setTextColor(255, 255, 255);
      doc.setFont("helvetica", "bold");
      doc.setFontSize(23);
      doc.text("QARBA", margin, 15);

      doc.setFontSize(12);
      doc.text("Advertising Forecast Report", margin, 25);

      doc.setFont("helvetica", "normal");
      doc.setFontSize(8);
      doc.text(
        `Generated: ${new Date().toLocaleDateString("en-NG")}`,
        pageWidth - margin,
        15,
        { align: "right" }
      );

      // Campaign summary.
      let y = 49;

      doc.setTextColor(30, 41, 59);
      doc.setFont("helvetica", "bold");
      doc.setFontSize(13);
      doc.text("Campaign details", margin, y);

      y += 9;
      doc.setFont("helvetica", "normal");
      doc.setFontSize(10);

      const details = [
        `Campaign: ${name.trim() || "Qarba Campaign"}`,
        `Budget: ${money(budget)}`,
        `Duration: ${days} days`,
        `Audience: ${audienceLabel(audience)}`,
        `Target location: ${locationLabel(targetLocation, selectedCountries)}`,
        `Daily budget: ${money(budget / days)}`,
      ];

      details.forEach((detail) => {
        const wrapped = doc.splitTextToSize(
          detail,
          pageWidth - margin * 2
        );

        doc.text(wrapped, margin, y);
        y += wrapped.length * 5 + 2;
      });

      // Indicate which calculation produced the displayed forecast.
      doc.setFontSize(8);
      doc.setTextColor(71, 85, 105);
      doc.text(
        backendResults
          ? "Forecast source: FastAPI backend"
          : "Forecast source: local dashboard calculation",
        margin,
        y + 2
      );

      // Comparison cards for the three forecast scenarios.
      const cardY = y + 10;
      const cardWidth = (pageWidth - margin * 2 - 12) / 3;
      const cardHeight = 79;
      const cardXs = [
        margin,
        margin + cardWidth + 6,
        margin + (cardWidth + 6) * 2,
      ];

      forecasts.forEach(({ label, forecast }, index) => {
        const x = cardXs[index];

        doc.setDrawColor(203, 213, 225);
        doc.setFillColor(248, 250, 252);
        doc.roundedRect(x, cardY, cardWidth, cardHeight, 3, 3, "FD");

        doc.setTextColor(30, 64, 175);
        doc.setFont("helvetica", "bold");
        doc.setFontSize(9);
        doc.text(label, x + 3, cardY + 8);

        const metrics: [string, string][] = [
          ["Impressions", number(forecast.impressions)],
          ["Link clicks", number(forecast.estimated_link_clicks)],
          ["Enquiries", number(forecast.estimated_enquiries)],
          ["Cost per click", money(forecast.cost_per_click)],
          ["Cost per enquiry", money(forecast.cost_per_enquiry)],
        ];

        metrics.forEach(([metricLabel, metricValue], metricIndex) => {
          const metricY = cardY + 17 + metricIndex * 12;

          doc.setFont("helvetica", "normal");
          doc.setFontSize(7);
          doc.setTextColor(71, 85, 105);
          doc.text(metricLabel, x + 3, metricY);

          doc.setFont("helvetica", "bold");
          doc.setFontSize(7.5);
          doc.setTextColor(15, 23, 42);
          doc.text(
            doc.splitTextToSize(metricValue, cardWidth - 6),
            x + 3,
            metricY + 4
          );
        });
      });

      // Assumptions table.
      let tableY = cardY + cardHeight + 12;

      if (tableY > pageHeight - 60) {
        doc.addPage();
        tableY = 22;
      }

      doc.setTextColor(30, 41, 59);
      doc.setFont("helvetica", "bold");
      doc.setFontSize(12);
      doc.text("Forecast assumptions", margin, tableY);

      tableY += 8;

      doc.setFillColor(239, 246, 255);
      doc.rect(margin, tableY - 5, pageWidth - margin * 2, 9, "F");

      doc.setFontSize(8);
      doc.text("Scenario", margin + 3, tableY);
      doc.text("CPM (NGN)", 76, tableY);
      doc.text("CTR (%)", 112, tableY);
      doc.text("Enquiry rate (%)", 145, tableY);

      forecasts.forEach(({ key, label }, index) => {
        const rowY = tableY + 9 + index * 8;
        const assumption = scenarios[key];

        doc.setFont("helvetica", "normal");
        doc.setFontSize(8);
        doc.setTextColor(51, 65, 85);
        doc.text(label, margin + 3, rowY);
        doc.text(number(assumption.cpm), 76, rowY);
        doc.text(number(assumption.ctr), 112, rowY);
        doc.text(number(assumption.enquiryRate), 145, rowY);
      });

      // Include the single estimate note requested for the report.
      let notesY = tableY + 9 + forecasts.length * 8 + 9;

      if (notesY > pageHeight - 25) {
        doc.addPage();
        notesY = 25;
      }

      doc.setDrawColor(203, 213, 225);
      doc.line(margin, notesY, pageWidth - margin, notesY);

      notesY += 9;
      doc.setFont("helvetica", "normal");
      doc.setFontSize(9);
      doc.setTextColor(71, 85, 105);

      const note =
        "All figures are estimates. Actual results may be better or worse depending on campaign performance.";

      doc.text(
        doc.splitTextToSize(note, pageWidth - margin * 2),
        margin,
        notesY
      );

      // Add a footer to every report page.
      const pageCount = doc.getNumberOfPages();

      for (let page = 1; page <= pageCount; page++) {
        doc.setPage(page);
        doc.setFont("helvetica", "normal");
        doc.setFontSize(8);
        doc.setTextColor(100, 116, 139);
        doc.text(
          "Qarba Advertising Planner",
          pageWidth / 2,
          pageHeight - 8,
          { align: "center" }
        );
      }

      // Generate a safe filename for the downloaded report.
      const safeName =
        name
          .trim()
          .replace(/[^a-z0-9-_]+/gi, "-")
          .replace(/^-+|-+$/g, "") || "qarba-campaign";

      doc.save(`${safeName}-report.pdf`);

      setMessage("PDF report generated successfully.");
      setError("");
    } catch (err) {
      setError(
        err instanceof Error
          ? `Could not generate PDF: ${err.message}`
          : "Could not generate the PDF report."
      );
    }
  };

  return (
    <main className="min-h-screen bg-slate-50 text-slate-900">
      {/* Dashboard header */}
      <header className="border-b border-slate-200 bg-white">
        <div className="mx-auto flex max-w-7xl items-center justify-between px-6 py-5">
          <div>
            <h1 className="text-2xl font-bold text-blue-800">QARBA</h1>
            <p className="text-sm text-slate-500">Advertising Planner</p>
          </div>

          <span className="rounded-full bg-blue-50 px-4 py-2 text-sm text-blue-800">
            Internal marketing tool
          </span>
        </div>
      </header>

      <div className="mx-auto max-w-7xl px-6 py-8">
        {/* Page introduction */}
        <section className="mb-8">
          <h2 className="text-3xl font-bold tracking-tight">
            Plan your next campaign
          </h2>
          <p className="mt-2 text-slate-600">
            Compare advertising budgets, locations, and possible outcomes.
          </p>
        </section>

        {/* Campaign settings and forecast comparison */}
        <section className="grid gap-6 lg:grid-cols-3">
          <div className="rounded-2xl border border-slate-200 bg-white p-6 shadow-sm">
            <h3 className="text-lg font-semibold">Campaign settings</h3>

            <label className="mt-5 block text-sm font-medium">
              Campaign name
              <input
                type="text"
                maxLength={150}
                value={name}
                onChange={(event) => {
                  setName(event.target.value);
                  invalidateResults();
                }}
                className="mt-2 w-full rounded-lg border border-slate-300 p-3 outline-none focus:border-blue-600"
              />
            </label>

            <label className="mt-5 block text-sm font-medium">
              Total budget (NGN)
              <input
                type="number"
                min="1"
                max="100000000"
                value={budget}
                onChange={(event) => {
                  const rawValue = event.target.value;
                  const parsed = Number(rawValue);

                  setBudget(
                    rawValue === ""
                      ? 1
                      : Math.min(
                          100000000,
                          Math.max(1, Number.isFinite(parsed) ? parsed : 1)
                        )
                  );
                  invalidateResults();
                }}
                className="mt-2 w-full rounded-lg border border-slate-300 p-3 outline-none focus:border-blue-600"
              />
            </label>

            <label className="mt-5 block text-sm font-medium">
              Campaign duration (days)
              <input
                type="number"
                min="1"
                max="365"
                value={days}
                onChange={(event) => {
                  const rawValue = event.target.value;
                  const parsed = Number(rawValue);

                  setDays(
                    rawValue === ""
                      ? 1
                      : Math.min(
                          365,
                          Math.max(1, Number.isFinite(parsed) ? parsed : 1)
                        )
                  );
                  invalidateResults();
                }}
                className="mt-2 w-full rounded-lg border border-slate-300 p-3 outline-none focus:border-blue-600"
              />
            </label>

            <label className="mt-5 block text-sm font-medium">
              Target audience
              <select
                value={audience}
                onChange={(event) => {
                  setAudience(event.target.value);
                  invalidateResults();
                }}
                className="mt-2 w-full rounded-lg border border-slate-300 p-3 outline-none focus:border-blue-600"
              >
                <option value="buyers">Property buyers</option>
                <option value="tenants">Tenants</option>
                <option value="both">Buyers and tenants</option>
              </select>
            </label>

            {/* Geographic targeting selector */}
            <label className="mt-5 block text-sm font-medium">
              Target location
              <select
                value={targetLocation}
                onChange={(event) => {
                  const location = event.target.value;

                  if (isLocationKey(location)) {
                    setTargetLocation(location);
                    invalidateResults();
                  }
                }}
                className="mt-2 w-full rounded-lg border border-slate-300 p-3 outline-none focus:border-blue-600"
              >
                <option value="nigeria">Nigeria only</option>
                <option value="selected_countries">Selected countries</option>
                <option value="worldwide">Worldwide</option>
              </select>
            </label>

            {targetLocation === "selected_countries" && (
              <label className="mt-4 block text-sm font-medium">
                Countries to target
                <input
                  type="text"
                  maxLength={500}
                  value={selectedCountries}
                  placeholder="e.g. Nigeria, Ghana, United Kingdom"
                  onChange={(event) => {
                    setSelectedCountries(event.target.value);
                    invalidateResults();
                  }}
                  className="mt-2 w-full rounded-lg border border-slate-300 p-3 outline-none focus:border-blue-600"
                />
              </label>
            )}

            <div className="mt-6 rounded-xl bg-blue-50 p-4">
              <p className="text-sm text-blue-800">Estimated daily budget</p>
              <p className="mt-1 text-2xl font-bold text-blue-900">
                {money(budget / days)}
              </p>
            </div>
          </div>

          {/* Forecast comparison cards */}
          <div className="lg:col-span-2">
            <h3 className="mb-2 text-xl font-semibold">
              Forecast comparison
            </h3>
            <p className="mb-4 text-sm text-slate-600">
              Targeting: {locationLabel(targetLocation, selectedCountries)}
            </p>

            <div className="grid gap-4 md:grid-cols-3">
              {forecasts.map(({ key, label, forecast }) => (
                <article
                  key={key}
                  className={`rounded-2xl border bg-white p-5 shadow-sm ${
                    key === "expected"
                      ? "border-blue-300 ring-1 ring-blue-100"
                      : "border-slate-200"
                  }`}
                >
                  <p className="font-semibold">{label}</p>

                  <p className="mt-4 text-sm text-slate-500">
                    Estimated impressions
                  </p>
                  <p className="text-2xl font-bold">
                    {number(forecast.impressions)}
                  </p>

                  <div className="mt-4 border-t border-slate-100 pt-4">
                    <p className="text-sm text-slate-500">Link clicks</p>
                    <p className="text-xl font-semibold">
                      {number(forecast.estimated_link_clicks)}
                    </p>
                  </div>

                  <div className="mt-4">
                    <p className="text-sm text-slate-500">
                      Estimated enquiries
                    </p>
                    <p className="text-xl font-semibold">
                      {number(forecast.estimated_enquiries)}
                    </p>
                  </div>

                  <div className="mt-4 border-t border-slate-100 pt-4">
                    <p className="text-sm text-slate-500">Cost per click</p>
                    <p className="font-semibold">
                      {money(forecast.cost_per_click)}
                    </p>
                  </div>

                  <div className="mt-3">
                    <p className="text-sm text-slate-500">Cost per enquiry</p>
                    <p className="font-semibold">
                      {money(forecast.cost_per_enquiry)}
                    </p>
                  </div>
                </article>
              ))}
            </div>

            <p className="mt-4 text-sm text-slate-500">
              All figures are estimates. Actual results may be better or worse
              depending on campaign performance.
            </p>
          </div>
        </section>

        {/* Forecast, save, and PDF actions */}
        <section className="mt-8 rounded-2xl border border-slate-200 bg-white p-6 shadow-sm">
          <h3 className="text-xl font-semibold">Campaign actions</h3>
          <p className="mt-2 text-sm text-slate-600">
            Calculate a forecast, save the campaign, or download a PDF report.
          </p>

          <div className="mt-4 flex flex-wrap gap-3">
            <button
              onClick={runForecast}
              disabled={loading || saving}
              className="rounded-lg border border-blue-700 px-5 py-3 font-medium text-blue-700 hover:bg-blue-50 disabled:opacity-50"
            >
              {loading ? "Calculating..." : "Run backend forecast"}
            </button>

            <button
              onClick={saveCampaign}
              disabled={loading || saving}
              className="rounded-lg bg-blue-700 px-5 py-3 font-medium text-white hover:bg-blue-800 disabled:opacity-50"
            >
              {saving ? "Saving..." : "Save campaign"}
            </button>

            <button
              onClick={downloadPdfReport}
              className="rounded-lg bg-slate-800 px-5 py-3 font-medium text-white hover:bg-slate-900"
            >
              Download PDF Report
            </button>
          </div>

          {message && (
            <p
              role="status"
              className="mt-4 rounded-lg bg-green-50 p-3 text-sm text-green-800"
            >
              {message}
            </p>
          )}

          {error && (
            <p
              role="alert"
              className="mt-4 rounded-lg bg-red-50 p-3 text-sm text-red-800"
            >
              {error}
            </p>
          )}
        </section>

        {/* Editable assumptions for the selected location */}
        <section className="mt-8 rounded-2xl border border-slate-200 bg-white p-6 shadow-sm">
          <h3 className="text-xl font-semibold">Forecast assumptions</h3>
          <p className="mt-2 text-sm text-slate-600">
            Edit the assumptions for{" "}
            {locationLabel(targetLocation, selectedCountries)}. Each location
            has its own editable assumptions.
          </p>

          <div className="mt-6 overflow-x-auto">
            <table className="w-full min-w-[650px] text-left text-sm">
              <thead>
                <tr className="border-b border-slate-200 text-slate-500">
                  <th className="pb-3 pr-4 font-medium">Scenario</th>
                  <th className="px-3 pb-3 font-medium">CPM (NGN)</th>
                  <th className="px-3 pb-3 font-medium">CTR (%)</th>
                  <th className="pb-3 pl-3 font-medium">Enquiry rate (%)</th>
                </tr>
              </thead>

              <tbody>
                {(
                  Object.entries(scenarios) as [ScenarioKey, Scenario][]
                ).map(([key, scenario]) => (
                  <tr key={key} className="border-b border-slate-100">
                    <td className="py-4 pr-4 font-medium">
                      {scenarioNames[key]}
                    </td>

                    {(
                      [
                        ["cpm", scenario.cpm],
                        ["ctr", scenario.ctr],
                        ["enquiryRate", scenario.enquiryRate],
                      ] as [keyof Scenario, number][]
                    ).map(([field, value]) => (
                      <td key={field} className="px-3 py-3">
                        <input
                          type="number"
                          min={field === "cpm" ? "0.01" : "0"}
                          max={field === "cpm" ? undefined : "100"}
                          step="any"
                          value={Number.isFinite(value) ? value : 0}
                          onChange={(event) => {
                            const rawValue = event.target.value;
                            const nextValue = Number(rawValue);

                            // Keep CPM positive and rates between 0 and 100.
                            const safeValue =
                              rawValue === ""
                                ? 0
                                : Math.max(
                                    0,
                                    Number.isFinite(nextValue) ? nextValue : 0
                                  );

                            updateScenario(key, field, safeValue);
                          }}
                          className="w-28 rounded-md border border-slate-300 p-2 outline-none focus:border-blue-600"
                        />
                      </td>
                    ))}
                  </tr>
                ))}
              </tbody>
            </table>
          </div>

          <p className="mt-4 text-xs text-slate-500">
            CPM is the cost per 1,000 impressions. CTR estimates the percentage
            of impressions that generate link clicks. Enquiry rate estimates
            the percentage of clicks that generate enquiries.
          </p>
        </section>

        {/* Saved campaign history */}
        <section className="mt-10 rounded-2xl border border-slate-200 bg-white p-6 shadow-sm">
          <div className="flex flex-wrap items-center justify-between gap-3">
            <div>
              <h3 className="text-xl font-semibold">Saved campaigns</h3>
              <p className="mt-1 text-sm text-slate-500">
                Campaigns stored in your SQLite database.
              </p>
            </div>

            <button
              onClick={() => void loadCampaigns()}
              disabled={listLoading}
              className="rounded-lg border border-slate-300 px-4 py-2 text-sm font-medium hover:bg-slate-50 disabled:opacity-50"
            >
              {listLoading ? "Refreshing..." : "Refresh campaigns"}
            </button>
          </div>

          <div className="mt-6 overflow-x-auto">
            <table className="w-full min-w-[850px] text-left text-sm">
              <thead>
                <tr className="border-b border-slate-200 text-slate-500">
                  <th className="pb-3 pr-4 font-medium">Campaign</th>
                  <th className="px-3 pb-3 font-medium">Budget</th>
                  <th className="px-3 pb-3 font-medium">Duration</th>
                  <th className="px-3 pb-3 font-medium">Audience</th>
                  <th className="px-3 pb-3 font-medium">Location</th>
                  <th className="pb-3 pl-3 font-medium">Action</th>
                </tr>
              </thead>

              <tbody>
                {savedCampaigns.map((campaign) => (
                  <tr key={campaign.id} className="border-b border-slate-100">
                    <td className="py-4 pr-4">
                      <p className="font-medium">{campaign.name}</p>
                      <p className="text-xs text-slate-500">
                        Campaign #{campaign.id}
                      </p>
                    </td>

                    <td className="px-3 py-4">{money(campaign.budget)}</td>

                    <td className="px-3 py-4">
                      {campaign.duration_days} days
                    </td>

                    <td className="px-3 py-4">
                      {audienceLabel(campaign.audience)}
                    </td>

                    <td className="px-3 py-4">
                      {locationLabel(
                        isLocationKey(campaign.target_location)
                          ? campaign.target_location
                          : "nigeria",
                        campaign.selected_countries ?? ""
                      )}
                    </td>

                    <td className="py-4 pl-3">
                      <button
                        onClick={() => viewCampaign(campaign)}
                        className="font-medium text-blue-700 hover:underline"
                      >
                        View forecast
                      </button>
                    </td>
                  </tr>
                ))}

                {savedCampaigns.length === 0 && !listLoading && (
                  <tr>
                    <td
                      colSpan={6}
                      className="py-8 text-center text-slate-500"
                    >
                      No saved campaigns yet. Save your first campaign above.
                    </td>
                  </tr>
                )}
              </tbody>
            </table>
          </div>
        </section>

        {/* Dashboard footer */}
        <footer className="py-8 text-center text-xs text-slate-500">
          Qarba Advertising Planner
        </footer>
      </div>
    </main>
  );
}
