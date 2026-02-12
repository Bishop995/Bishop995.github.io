# Bishop995.github.io
import React, { useState, useEffect } from 'react';

/**
 * VINYL RECORD TRACKER
 * 
 * This app helps you track vinyl records you own and want to buy.
 * 
 * KEY FEATURES:
 * - Toggle between "Have" and "Want" collections
 * - Search for albums and add them to your lists
 * - View album artwork
 * - Everything saves automatically (persists between sessions)
 * - Different color schemes for each list
 */

export default function VinylTracker() {
  // ============================================
  // STATE MANAGEMENT
  // ============================================
  // Think of "state" as the app's memory - it remembers things like:
  // - Which view you're on (Have or Want)
  // - What records you've added
  // - What you're searching for
  
  const [currentView, setCurrentView] = useState('have'); // 'have' or 'want'
  const [haveRecords, setHaveRecords] = useState([]);
  const [wantRecords, setWantRecords] = useState([]);
  const [searchQuery, setSearchQuery] = useState('');
  const [searchResults, setSearchResults] = useState([]);
  const [isSearching, setIsSearching] = useState(false);
  const [isLoading, setIsLoading] = useState(true);
  const [predictions, setPredictions] = useState([]);
  const [showPredictions, setShowPredictions] = useState(false);

  // ============================================
  // LOAD SAVED DATA WHEN APP STARTS
  // ============================================
  // This runs once when the app first loads
  // It retrieves your saved collections from storage
  
  useEffect(() => {
    loadCollections();
  }, []);

  async function loadCollections() {
    try {
      // Try to load the "have" collection from storage
      const haveResult = await window.storage.get('vinyl-have');
      if (haveResult?.value) {
        setHaveRecords(JSON.parse(haveResult.value));
      }

      // Try to load the "want" collection from storage
      const wantResult = await window.storage.get('vinyl-want');
      if (wantResult?.value) {
        setWantRecords(JSON.parse(wantResult.value));
      }
    } catch (error) {
      console.log('No saved collections yet');
    } finally {
      setIsLoading(false);
    }
  }

  // ============================================
  // SAVE DATA WHENEVER IT CHANGES
  // ============================================
  // These run whenever you add/remove records
  // They automatically save your collections
  
  useEffect(() => {
    if (!isLoading && haveRecords.length >= 0) {
      saveCollection('vinyl-have', haveRecords);
    }
  }, [haveRecords, isLoading]);

  useEffect(() => {
    if (!isLoading && wantRecords.length >= 0) {
      saveCollection('vinyl-want', wantRecords);
    }
  }, [wantRecords, isLoading]);

  async function saveCollection(key, data) {
    try {
      await window.storage.set(key, JSON.stringify(data));
    } catch (error) {
      console.error('Error saving collection:', error);
    }
  }

  // ============================================
  // GENERATE SEARCH PREDICTIONS
  // ============================================
  // This runs as you type and suggests albums/artists
  
  async function generatePredictions(query) {
    if (!query.trim() || query.length < 2) {
      setPredictions([]);
      setShowPredictions(false);
      return;
    }

    try {
      const response = await fetch('https://api.anthropic.com/v1/messages', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({
          model: 'claude-sonnet-4-20250514',
          max_tokens: 500,
          messages: [{
            role: 'user',
            content: `Based on the partial search "${query}", suggest 5 likely album or artist completions. Return ONLY a JSON array of strings (no markdown, no explanation):
["suggestion 1", "suggestion 2", "suggestion 3", "suggestion 4", "suggestion 5"]

Examples should be real, popular albums or artists that match the query.`
          }]
        })
      });

      const data = await response.json();
      const content = data.content.find(item => item.type === 'text')?.text || '';
      const cleanContent = content.replace(/```json\n?/g, '').replace(/```\n?/g, '').trim();
      const suggestions = JSON.parse(cleanContent);
      
      setPredictions(suggestions);
      setShowPredictions(true);
    } catch (error) {
      console.error('Prediction error:', error);
    }
  }

  // Debounce predictions so we don't call the API on every keystroke
  useEffect(() => {
    const timer = setTimeout(() => {
      if (searchQuery) {
        generatePredictions(searchQuery);
      }
    }, 500); // Wait 500ms after user stops typing

    return () => clearTimeout(timer);
  }, [searchQuery]);

  // Close predictions when clicking outside
  useEffect(() => {
    function handleClickOutside(event) {
      // Check if click is outside the search area
      const searchContainer = event.target.closest('[data-search-container]');
      if (!searchContainer && showPredictions) {
        setShowPredictions(false);
      }
    }

    document.addEventListener('mousedown', handleClickOutside);
    return () => {
      document.removeEventListener('mousedown', handleClickOutside);
    };
  }, [showPredictions]);

  // ============================================
  // SEARCH FOR ALBUMS
  // ============================================
  // This function searches for albums with real artwork using web search
  
  async function searchAlbums() {
    if (!searchQuery.trim()) return;
    
    setIsSearching(true);
    setSearchResults([]);
    setShowPredictions(false); // Close predictions when searching
    setPredictions([]); // Clear predictions

    try {
      // Use AI with web_search tool to find albums with real cover art
      const response = await fetch('https://api.anthropic.com/v1/messages', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({
          model: 'claude-sonnet-4-20250514',
          max_tokens: 2000,
          tools: [{
            type: "web_search_20250305",
            name: "web_search"
          }],
          messages: [{
            role: 'user',
            content: `Search for album "${searchQuery}" and find up to 5 matching albums. For each album, search the web to find a direct link to the album cover image (look for JPG, PNG, or WebP URLs from music sites like last.fm, discogs, or musicbrainz).

Return ONLY a JSON array with this format:
[
  {
    "artist": "Artist Name",
    "album": "Album Title",
    "year": "Year",
    "imageUrl": "https://direct-link-to-album-cover.jpg"
  }
]

Make sure imageUrl is a DIRECT link to an image file (ending in .jpg, .png, .webp, etc), not a webpage.`
          }]
        })
      });

      const data = await response.json();
      
      // Extract text from all content blocks
      let textContent = '';
      for (const block of data.content) {
        if (block.type === 'text') {
          textContent += block.text + '\n';
        }
      }
      
      // Clean up the response and parse it as JSON
      const cleanContent = textContent.replace(/```json\n?/g, '').replace(/```\n?/g, '').trim();
      
      // Find JSON array in the content
      const jsonMatch = cleanContent.match(/\[\s*{[\s\S]*}\s*\]/);
      if (jsonMatch) {
        const results = JSON.parse(jsonMatch[0]);
        setSearchResults(results);
      } else {
        throw new Error('No results found');
      }
    } catch (error) {
      console.error('Search error:', error);
      alert('Search failed. Please try again.');
    } finally {
      setIsSearching(false);
    }
  }

  // ============================================
  // ADD ALBUM TO COLLECTION
  // ============================================
  // When you click "Add to Have" or "Add to Want"
  
  function addToCollection(album, collection) {
    const newRecord = {
      ...album,
      id: Date.now(), // Unique ID for this record
      addedDate: new Date().toISOString()
    };

    if (collection === 'have') {
      setHaveRecords([...haveRecords, newRecord]);
    } else {
      setWantRecords([...wantRecords, newRecord]);
    }

    // Clear search after adding
    setSearchResults([]);
    setSearchQuery('');
  }

  // ============================================
  // REMOVE ALBUM FROM COLLECTION
  // ============================================
  
  function removeFromCollection(id, collection) {
    if (collection === 'have') {
      setHaveRecords(haveRecords.filter(record => record.id !== id));
    } else {
      setWantRecords(wantRecords.filter(record => record.id !== id));
    }
  }

  // ============================================
  // DETERMINE CURRENT COLORS
  // ============================================
  // Different color schemes for Have vs Want
  
  const colors = currentView === 'have' 
    ? {
        primary: '#10b981',      // Green
        secondary: '#065f46',    // Deep green
        accent: '#34d399',       // Light green
        bg: '#064e3b',          // Dark green
        cardBg: '#065f46',      // Card background
        text: '#d1fae5'         // Light green text
      }
    : {
        primary: '#3b82f6',     // Blue
        secondary: '#1e40af',   // Deep blue
        accent: '#60a5fa',      // Light blue
        bg: '#1e3a8a',         // Dark blue
        cardBg: '#1e40af',     // Card background
        text: '#dbeafe'        // Light blue text
      };

  // Get the current collection based on view
  const currentRecords = currentView === 'have' ? haveRecords : wantRecords;

  // ============================================
  // RENDER THE USER INTERFACE
  // ============================================
  
  return (
    <div style={{
      minHeight: '100vh',
      background: `linear-gradient(135deg, ${colors.bg} 0%, ${colors.cardBg} 100%)`,
      color: colors.text,
      fontFamily: "'Courier New', monospace",
      transition: 'all 0.5s ease',
      padding: '20px'
    }}>
      
      {/* ===== HEADER WITH TOGGLE ===== */}
      <div style={{
        maxWidth: '1200px',
        margin: '0 auto',
        marginBottom: '30px'
      }}>
        {/* THE TOGGLE BUTTON */}
        <div style={{
          display: 'flex',
          justifyContent: 'center',
          gap: '0',
          marginBottom: '30px',
          marginTop: '20px'
        }}>
          <button
            onClick={() => setCurrentView('have')}
            style={{
              padding: '15px 40px',
              fontSize: '1.2rem',
              fontFamily: "'Courier New', monospace",
              fontWeight: 'bold',
              background: currentView === 'have' ? '#10b981' : '#4b5563',
              color: currentView === 'have' ? '#d1fae5' : '#9ca3af',
              border: 'none',
              borderRadius: '8px 0 0 8px',
              cursor: 'pointer',
              transition: 'all 0.3s ease',
              transform: currentView === 'have' ? 'scale(1.05)' : 'scale(1)',
              boxShadow: currentView === 'have' ? '0 4px 12px rgba(16, 185, 129, 0.4)' : 'none'
            }}
          >
            HAVE ({haveRecords.length})
          </button>
          <button
            onClick={() => setCurrentView('want')}
            style={{
              padding: '15px 40px',
              fontSize: '1.2rem',
              fontFamily: "'Courier New', monospace",
              fontWeight: 'bold',
              background: currentView === 'want' ? '#3b82f6' : '#4b5563',
              color: currentView === 'want' ? '#dbeafe' : '#9ca3af',
              border: 'none',
              borderRadius: '0 8px 8px 0',
              cursor: 'pointer',
              transition: 'all 0.3s ease',
              transform: currentView === 'want' ? 'scale(1.05)' : 'scale(1)',
              boxShadow: currentView === 'want' ? '0 4px 12px rgba(59, 130, 246, 0.4)' : 'none'
            }}
          >
            WANT ({wantRecords.length})
          </button>
        </div>

        {/* ===== SEARCH BAR ===== */}
        <div style={{
          background: colors.cardBg,
          padding: '25px',
          borderRadius: '12px',
          border: `2px solid ${colors.primary}`,
          boxShadow: `0 8px 16px rgba(0,0,0,0.3)`
        }}>
          <h2 style={{
            margin: '0 0 15px 0',
            color: colors.accent,
            fontSize: '1.3rem'
          }}>
            Search for Albums
          </h2>
          <div style={{ position: 'relative' }} data-search-container="true">
            <div style={{ display: 'flex', gap: '10px' }}>
              <input
                type="text"
                value={searchQuery}
                onChange={(e) => {
                  setSearchQuery(e.target.value);
                  if (!e.target.value.trim()) {
                    setShowPredictions(false);
                  }
                }}
                onKeyPress={(e) => {
                  if (e.key === 'Enter') {
                    searchAlbums();
                    setShowPredictions(false);
                  }
                }}
                onFocus={() => {
                  if (predictions.length > 0) {
                    setShowPredictions(true);
                  }
                }}
                placeholder="Enter artist name or album title..."
                style={{
                  flex: 1,
                  padding: '12px',
                  fontSize: '1rem',
                  background: colors.bg,
                  border: `2px solid ${colors.secondary}`,
                  borderRadius: '6px',
                  color: colors.text,
                  fontFamily: "'Courier New', monospace"
                }}
              />
              <button
                onClick={() => {
                  searchAlbums();
                  setShowPredictions(false);
                }}
                disabled={isSearching}
                style={{
                  padding: '12px 30px',
                  fontSize: '1rem',
                  fontWeight: 'bold',
                  background: colors.primary,
                  color: colors.bg,
                  border: 'none',
                  borderRadius: '6px',
                  cursor: isSearching ? 'wait' : 'pointer',
                  fontFamily: "'Courier New', monospace",
                  opacity: isSearching ? 0.6 : 1
                }}
              >
                {isSearching ? 'SEARCHING...' : 'SEARCH'}
              </button>
            </div>

            {/* PREDICTION DROPDOWN */}
            {showPredictions && predictions.length > 0 && (
              <div style={{
                position: 'absolute',
                top: '100%',
                left: 0,
                right: '150px',
                marginTop: '5px',
                background: colors.bg,
                border: `2px solid ${colors.primary}`,
                borderRadius: '6px',
                maxHeight: '200px',
                overflowY: 'auto',
                zIndex: 1000,
                boxShadow: '0 4px 12px rgba(0,0,0,0.4)'
              }}>
                {predictions.map((prediction, index) => (
                  <div
                    key={index}
                    onClick={() => {
                      setSearchQuery(prediction);
                      setShowPredictions(false);
                    }}
                    style={{
                      padding: '12px',
                      cursor: 'pointer',
                      borderBottom: index < predictions.length - 1 ? `1px solid ${colors.secondary}` : 'none',
                      transition: 'background 0.2s ease'
                    }}
                    onMouseEnter={(e) => e.currentTarget.style.background = colors.secondary}
                    onMouseLeave={(e) => e.currentTarget.style.background = 'transparent'}
                  >
                    {prediction}
                  </div>
                ))}
              </div>
            )}
          </div>

          {/* ===== SEARCH RESULTS ===== */}
          {searchResults.length > 0 && (
            <div style={{ marginTop: '20px' }}>
              <h3 style={{ color: colors.accent, marginBottom: '12px' }}>Results:</h3>
              <div style={{ display: 'grid', gap: '12px' }}>
                {searchResults.map((result, index) => (
                  <div
                    key={index}
                    style={{
                      background: colors.bg,
                      padding: '12px',
                      borderRadius: '8px',
                      border: `1px solid ${colors.secondary}`,
                      display: 'flex',
                      justifyContent: 'space-between',
                      alignItems: 'center',
                      gap: '12px'
                    }}
                  >
                    {/* Album artwork thumbnail */}
                    {result.imageUrl && (
                      <img 
                        src={result.imageUrl} 
                        alt={result.album}
                        style={{
                          width: '50px',
                          height: '50px',
                          borderRadius: '6px',
                          objectFit: 'cover',
                          flexShrink: 0
                        }}
                      />
                    )}
                    <div style={{ flex: 1, minWidth: 0 }}>
                      <div style={{ 
                        fontWeight: 'bold', 
                        fontSize: '1rem', 
                        marginBottom: '4px',
                        overflow: 'hidden',
                        textOverflow: 'ellipsis',
                        whiteSpace: 'nowrap'
                      }}>
                        {result.album}
                      </div>
                      <div style={{ 
                        color: colors.accent,
                        fontSize: '0.9rem',
                        overflow: 'hidden',
                        textOverflow: 'ellipsis',
                        whiteSpace: 'nowrap'
                      }}>
                        {result.artist} • {result.year}
                      </div>
                    </div>
                    <button
                      onClick={() => addToCollection(result, currentView)}
                      style={{
                        padding: '6px 16px',
                        background: colors.primary,
                        color: colors.bg,
                        border: 'none',
                        borderRadius: '6px',
                        cursor: 'pointer',
                        fontFamily: "'Courier New', monospace",
                        fontWeight: 'bold',
                        fontSize: '0.85rem',
                        flexShrink: 0
                      }}
                    >
                      ADD
                    </button>
                  </div>
                ))}
              </div>
            </div>
          )}
        </div>
      </div>

      {/* ===== YOUR COLLECTION ===== */}
      <div style={{ maxWidth: '1200px', margin: '0 auto' }}>
        <h2 style={{
          fontSize: '2rem',
          marginBottom: '20px',
          color: colors.accent,
          textAlign: 'center',
          letterSpacing: '1px'
        }}>
          {currentView === 'have' ? '🎵 MY COLLECTION' : '🎯 WISHLIST'}
        </h2>

        {currentRecords.length === 0 ? (
          <div style={{
            background: colors.cardBg,
            padding: '60px',
            borderRadius: '12px',
            textAlign: 'center',
            border: `2px dashed ${colors.secondary}`,
            fontSize: '1.2rem',
            color: colors.accent
          }}>
            No records yet. Search and add some above!
          </div>
        ) : (
          <div style={{
            display: 'grid',
            gridTemplateColumns: 'repeat(auto-fill, minmax(400px, 1fr))',
            gap: '12px'
          }}>
            {currentRecords.map((record) => (
              <div
                key={record.id}
                style={{
                  background: colors.cardBg,
                  borderRadius: '8px',
                  padding: '12px',
                  border: `2px solid ${colors.primary}`,
                  boxShadow: `0 4px 12px rgba(0,0,0,0.3)`,
                  transition: 'transform 0.2s ease',
                  cursor: 'pointer',
                  display: 'flex',
                  alignItems: 'center',
                  gap: '12px'
                }}
                onMouseEnter={(e) => e.currentTarget.style.transform = 'translateY(-3px)'}
                onMouseLeave={(e) => e.currentTarget.style.transform = 'translateY(0)'}
              >
                {/* Left side: Album info */}
                <div style={{ flex: 1, minWidth: 0 }}>
                  <h3 style={{
                    fontSize: '1rem',
                    marginBottom: '4px',
                    color: colors.text,
                    fontWeight: 'bold',
                    overflow: 'hidden',
                    textOverflow: 'ellipsis',
                    whiteSpace: 'nowrap'
                  }}>
                    {record.album}
                  </h3>
                  <p style={{
                    color: colors.accent,
                    marginBottom: '4px',
                    fontSize: '0.9rem',
                    overflow: 'hidden',
                    textOverflow: 'ellipsis',
                    whiteSpace: 'nowrap'
                  }}>
                    {record.artist}
                  </p>
                  <p style={{
                    color: colors.secondary,
                    fontSize: '0.85rem',
                    marginBottom: '8px'
                  }}>
                    {record.year}
                  </p>

                  <button
                    onClick={() => removeFromCollection(record.id, currentView)}
                    style={{
                      padding: '6px 16px',
                      background: colors.secondary,
                      color: colors.text,
                      border: 'none',
                      borderRadius: '6px',
                      cursor: 'pointer',
                      fontFamily: "'Courier New', monospace",
                      fontWeight: 'bold',
                      fontSize: '0.85rem'
                    }}
                  >
                    REMOVE
                  </button>
                </div>

                {/* Right side: Album Art */}
                <div style={{
                  width: '80px',
                  height: '80px',
                  background: record.imageUrl ? 'transparent' : `linear-gradient(135deg, ${colors.secondary}, ${colors.primary})`,
                  borderRadius: '8px',
                  display: 'flex',
                  alignItems: 'center',
                  justifyContent: 'center',
                  fontSize: '2rem',
                  flexShrink: 0,
                  overflow: 'hidden'
                }}>
                  {record.imageUrl ? (
                    <img 
                      src={record.imageUrl} 
                      alt={record.album}
                      style={{
                        width: '100%',
                        height: '100%',
                        objectFit: 'cover'
                      }}
                    />
                  ) : (
                    '💿'
                  )}
                </div>
              </div>
            ))}
          </div>
        )}
      </div>
    </div>
  );
}
