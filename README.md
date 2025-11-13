import React, { useState, useCallback, useMemo } from 'react';
import { Search, Loader2, AlertTriangle, MessageSquare, ExternalLink } from 'lucide-react';

// --- Global Constants and Configuration ---

// These are global variables provided by the Canvas environment.
const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
const apiKey = ""; // API key is provided at runtime by the environment

const API_BASE_URL = "https://generativelanguage.googleapis.com/v1beta/models/";
const MODEL_NAME = "gemini-2.5-flash-preview-09-2025";
const API_URL = `${API_BASE_URL}${MODEL_NAME}:generateContent?key=${apiKey}`;

// --- Utility Functions ---

/**
 * Custom hook for exponential backoff retry mechanism.
 */
const useApiWithBackoff = () => {
  const [loading, setLoading] = useState(false);

  const fetchWithRetry = useCallback(async (payload, maxRetries = 5) => {
    setLoading(true);
    let lastError = null;

    for (let attempt = 0; attempt < maxRetries; attempt++) {
      try {
        if (attempt > 0) {
          const delay = Math.pow(2, attempt) * 1000 + Math.random() * 1000;
          await new Promise(resolve => setTimeout(resolve, delay));
        }

        const response = await fetch(API_URL, {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify(payload)
        });

        if (response.status === 429 || !response.ok) {
          if (response.status === 429) {
            lastError = new Error(`Rate limit exceeded. Attempt ${attempt + 1}.`);
          } else {
            const errorBody = await response.text();
            lastError = new Error(`API failed with status ${response.status}: ${errorBody}`);
            // For non-429 errors, we might break out early unless specifically designed to retry
            if (response.status < 500) break; 
          }
          continue; // Retry on 429 or 5xx errors
        }

        const result = await response.json();
        setLoading(false);
        return result;

      } catch (error) {
        lastError = error;
        console.error(`Attempt ${attempt + 1} failed:`, error);
      }
    }

    setLoading(false);
    throw lastError || new Error("All API attempts failed.");

  }, []);

  return { fetchWithRetry, loading };
};


// The original function declaration had a typo ('consconstt') which is now fixed to 'const'.
const checkCrisisKeywords = (text) => {
  if (!text) return false;
  const crisisKeywords = [
    'crisis', 'emergency', 'disaster', 'fire', 'flood', 'earthquake', 
    'attack', 'bomb', 'evacuate', 'hostage', 'casualty', 'fatality', 
    'lockdown', 'danger', 'urgent', 'alert', 'help now', 'rescue'
  ];
  const lowerText = text.toLowerCase();
  return crisisKeywords.some(keyword => lowerText.includes(keyword));
};

// --- Main App Component ---

const App = () => {
  const [query, setQuery] = useState('');
  const [responseText, setResponseText] = useState('');
  const [sources, setSources] = useState([]);
  const [isCrisis, setIsCrisis] = useState(false);

  const { fetchWithRetry, loading } = useApiWithBackoff();
  
  // System instruction to guide the model's behavior
  const systemPrompt = "You are an expert real-time information analyst. Your task is to concisely summarize the requested information based on current Google Search results. Prioritize factual and up-to-date data. Respond professionally and clearly.";

  const handleSearch = useCallback(async (e) => {
    e.preventDefault();
    if (loading || !query.trim()) return;

    setResponseText('');
    setSources([]);
    const crisisCheck = checkCrisisKeywords(query.trim());
    setIsCrisis(crisisCheck);

    const userQuery = `Search and summarize the current situation regarding: ${query.trim()}`;

    const payload = {
      contents: [{ parts: [{ text: userQuery }] }],
      tools: [{ "google_search": {} }],
      systemInstruction: { parts: [{ text: systemPrompt }] },
    };

    try {
      const result = await fetchWithRetry(payload);
      
      const candidate = result.candidates?.[0];
      const newText = candidate?.content?.parts?.[0]?.text || "No information found or generation failed.";
      
      let newSources = [];
      const groundingMetadata = candidate?.groundingMetadata;

      if (groundingMetadata && groundingMetadata.groundingAttributions) {
        newSources = groundingMetadata.groundingAttributions
          .map(attribution => ({
            uri: attribution.web?.uri,
            title: attribution.web?.title,
          }))
          .filter(source => source.uri && source.title);
      }

      setResponseText(newText);
      setSources(newSources);
      
    } catch (error) {
      console.error("API Call Error:", error);
      setResponseText(`An error occurred: ${error.message}. Please check the console for details.`);
      setSources([]);
    }
  }, [query, loading, fetchWithRetry]);

  // UI components
  const CrisisAlert = (
    <div className="bg-red-100 border-l-4 border-red-500 text-red-700 p-4 rounded-lg mb-6 shadow-md" role="alert">
      <div className="flex items-center">
        <AlertTriangle className="h-6 w-6 mr-3 text-red-500" />
        <p className="font-bold text-lg">Potential Crisis Alert</p>
      </div>
      <p className="mt-1 text-sm">The query contains keywords suggesting a potentially urgent or critical situation. Please verify information from official sources.</p>
    </div>
  );

  const SourcesDisplay = useMemo(() => (
    sources.length > 0 && (
      <div className="mt-6 p-4 bg-gray-50 border border-gray-200 rounded-xl">
        <h3 className="flex items-center text-sm font-semibold text-gray-700 mb-2">
          <MessageSquare className="h-4 w-4 mr-2" />
          Information Sources
        </h3>
        <ul className="space-y-2">
          {sources.map((source, index) => (
            <li key={index} className="text-xs">
              <a 
                href={source.uri} 
                target="_blank" 
                rel="noopener noreferrer" 
                className="text-blue-600 hover:text-blue-800 transition duration-150 ease-in-out flex items-start"
              >
                <ExternalLink className="h-3 w-3 mt-1 mr-1 flex-shrink-0" />
                <span className="break-all">{source.title || source.uri}</span>
              </a>
            </li>
          ))}
        </ul>
      </div>
    )
  ), [sources]);

  return (
    <div className="min-h-screen bg-gray-100 p-4 sm:p-8 font-['Inter']">
      <div className="max-w-3xl mx-auto">
        <header className="text-center mb-10">
          <h1 className="text-4xl font-extrabold text-gray-900 mb-2 tracking-tight">
            <span className="text-indigo-600">Gemini</span> Real-Time Analyst
          </h1>
          <p className="text-lg text-gray-600">Access grounded, up-to-date summaries using Google Search.</p>
        </header>

        {/* Search Form */}
        <form onSubmit={handleSearch} className="mb-8 relative shadow-xl rounded-2xl bg-white p-2">
          <div className="flex">
            <input
              type="text"
              value={query}
              onChange={(e) => setQuery(e.target.value)}
              placeholder="Search for current news, events, or information..."
              className="flex-grow p-4 text-gray-700 text-lg focus:ring-0 border-none outline-none bg-transparent rounded-l-2xl"
              disabled={loading}
              aria-label="Search query input"
            />
            <button
              type="submit"
              disabled={loading || !query.trim()}
              className={`p-4 rounded-xl font-semibold transition duration-300 ease-in-out flex items-center justify-center m-1
                ${loading || !query.trim()
                  ? 'bg-gray-300 text-gray-500 cursor-not-allowed'
                  : 'bg-indigo-600 hover:bg-indigo-700 text-white shadow-lg shadow-indigo-200'
                }`}
              aria-live="polite"
            >
              {loading ? (
                <>
                  <Loader2 className="h-5 w-5 animate-spin mr-2" />
                  Analyzing...
                </>
              ) : (
                <>
                  <Search className="h-5 w-5 mr-2" />
                  Search
                </>
              )}
            </button>
          </div>
        </form>

        {isCrisis && CrisisAlert}

        {/* Results Display */}
        <div className="bg-white p-6 sm:p-8 rounded-2xl shadow-xl min-h-[200px] border-t-4 border-indigo-500">
          <h2 className="text-2xl font-bold text-gray-900 mb-4">
            Response ({appId})
          </h2>
          
          {responseText ? (
            <>
              <div className="text-gray-800 leading-relaxed whitespace-pre-wrap">
                {responseText}
              </div>
              {SourcesDisplay}
            </>
          ) : (
            <div className="text-gray-500 text-center py-10">
              {loading ? (
                <div className="flex flex-col items-center">
                  <Loader2 className="h-8 w-8 text-indigo-500 animate-spin mb-3" />
                  <p>Fetching and synthesizing information...</p>
                </div>
              ) : (
                <p>Enter a query above to begin your real-time search.</p>
              )}
            </div>
          )}
        </div>
      </div>
    </div>
  );
};

export default App; 
