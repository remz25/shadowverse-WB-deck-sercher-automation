# shadowverse-WB-deck-sercher-automation
Multiple fallback selectors - tries different ways to find elements Enhanced error handling - graceful failures with retries Better popup handling - dismisses cookies/banners automatically Improved search terms - specifically targets Abyss craft content  🎯
"""

#!/usr/bin/env python3
"""
Shadowverse Abyss Craft Deck Finder
Searches YouTube for Abyss craft decks and creates deck list notes
Perfect for competitive card game research!
"""

from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from selenium.webdriver.chrome.options import Options
from selenium.webdriver.common.keys import Keys
import time
import logging
import json
import re
from datetime import datetime

# Setup logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger(__name__)

class ShadowverseAbyssFinder:
    def __init__(self, headless=False):
        """Initialize the browser driver"""
        self.driver = None
        self.wait = None
        self.deck_notes = []
        self.setup_driver(headless)
    
    def setup_driver(self, headless=False):
        """Setup Chrome WebDriver with enhanced options"""
        try:
            chrome_options = Options()
            if headless:
                chrome_options.add_argument("--headless")
            
            # Enhanced options for better YouTube compatibility
            chrome_options.add_argument("--no-sandbox")
            chrome_options.add_argument("--disable-dev-shm-usage")
            chrome_options.add_argument("--disable-blink-features=AutomationControlled")
            chrome_options.add_argument("--disable-extensions")
            chrome_options.add_experimental_option("excludeSwitches", ["enable-automation"])
            chrome_options.add_experimental_option('useAutomationExtension', False)
            chrome_options.add_argument("--window-size=1280,720")
            chrome_options.add_argument("--start-maximized")
            
            # More realistic user agent
            chrome_options.add_argument("--user-agent=Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36")
            
            self.driver = webdriver.Chrome(options=chrome_options)
            self.driver.implicitly_wait(5)
            self.wait = WebDriverWait(self.driver, 20)
            
            # Remove webdriver traces
            self.driver.execute_script("Object.defineProperty(navigator, 'webdriver', {get: () => undefined})")
            
            logger.info("✅ WebDriver initialized for Shadowverse search")
            return True
        except Exception as e:
            logger.error(f"❌ Failed to setup WebDriver: {e}")
            return False
    
    def navigate_to_youtube(self):
        """Navigate to YouTube with error handling"""
        try:
            logger.info("🌐 Navigating to YouTube...")
            self.driver.get("https://www.youtube.com")
            
            # Wait for main elements with multiple fallbacks
            try:
                # Try different possible search box selectors
                search_selectors = [
                    (By.NAME, "search_query"),
                    (By.ID, "search"),
                    (By.XPATH, "//input[@placeholder='Search']"),
                    (By.XPATH, "//form[@id='search-form']//input")
                ]
                
                search_found = False
                for selector_type, selector in search_selectors:
                    try:
                        search_element = self.wait.until(EC.presence_of_element_located((selector_type, selector)))
                        if search_element:
                            search_found = True
                            logger.info(f"✅ Found search box with selector: {selector}")
                            break
                    except:
                        continue
                
                if not search_found:
                    raise Exception("Could not find search box")
                
            except Exception as e:
                logger.error(f"❌ Search box not found: {e}")
                return False
            
            # Handle consent/cookie banners with multiple approaches
            self.handle_popups()
            
            logger.info("✅ Successfully navigated to YouTube")
            return True
            
        except Exception as e:
            logger.error(f"❌ Failed to navigate to YouTube: {e}")
            return False
    
    def handle_popups(self):
        """Handle various popups and banners"""
        try:
            time.sleep(2)  # Wait for popups to appear
            
            popup_selectors = [
                # Cookie/consent buttons
                "//button[contains(text(), 'Accept')]",
                "//button[contains(text(), 'I agree')]",
                "//button[contains(text(), 'Accept all')]",
                "//button[@aria-label='Accept all']",
                # Dismiss buttons
                "//button[contains(text(), 'Dismiss')]",
                "//button[contains(text(), 'No thanks')]",
                "//button[@aria-label='Dismiss']",
                # Close buttons
                "//button[@aria-label='Close']",
                "//button[contains(@class, 'close')]"
            ]
            
            for selector in popup_selectors:
                try:
                    popup_button = self.driver.find_element(By.XPATH, selector)
                    if popup_button.is_displayed():
                        popup_button.click()
                        logger.info(f"✅ Dismissed popup with: {selector}")
                        time.sleep(1)
                        break
                except:
                    continue
                    
        except Exception as e:
            logger.info("ℹ️ No popups to handle or already handled")
    
    def search_abyss_craft(self):
        """Search specifically for Abyss craft decks"""
        try:
            # Enhanced search terms for Abyss craft
            search_terms = [
                "shadowverse abyss craft deck",
                "shadowverse abyssal deck guide", 
                "shadowverse necromancer deck",
                "abyss craft shadowverse rotation"
            ]
            
            # Try the primary search term first
            primary_search = "shadowverse abyss craft deck 2024"
            
            logger.info(f"🔍 Searching for: {primary_search}")
            
            # Find search box with multiple selectors
            search_box = None
            search_selectors = [
                (By.NAME, "search_query"),
                (By.XPATH, "//input[@id='search']"),
                (By.XPATH, "//input[@placeholder='Search']")
            ]
            
            for selector_type, selector in search_selectors:
                try:
                    search_box = self.wait.until(EC.element_to_be_clickable((selector_type, selector)))
                    break
                except:
                    continue
            
            if not search_box:
                raise Exception("Could not find search box")
            
            # Clear and enter search term
            search_box.click()
            search_box.clear()
            time.sleep(1)
            search_box.send_keys(primary_search)
            logger.info(f"✅ Entered search term: {primary_search}")
            
            # Submit search
            search_box.send_keys(Keys.RETURN)
            
            # Wait for results with multiple indicators
            result_indicators = [
                (By.XPATH, "//div[@id='contents']"),
                (By.XPATH, "//ytd-video-renderer"),
                (By.XPATH, "//div[contains(@class, 'ytd-search')]")
            ]
            
            results_loaded = False
            for selector_type, selector in result_indicators:
                try:
                    self.wait.until(EC.presence_of_element_located((selector_type, selector)))
                    results_loaded = True
                    break
                except:
                    continue
            
            if results_loaded:
                logger.info("✅ Search results loaded successfully")
                time.sleep(3)  # Let results fully render
                return True
            else:
                raise Exception("Search results did not load")
                
        except Exception as e:
            logger.error(f"❌ Search failed: {e}")
            return False
    
    def extract_video_info(self, max_videos=8):
        """Extract video information with enhanced selectors"""
        try:
            logger.info("📹 Extracting video information...")
            
            # Wait for videos to load
            time.sleep(3)
            
            # Try different video container selectors
            video_selectors = [
                "//ytd-video-renderer",
                "//div[@class='ytd-video-renderer']",
                "//div[contains(@class, 'video-renderer')]"
            ]
            
            videos = []
            for selector in video_selectors:
                try:
                    video_elements = self.driver.find_elements(By.XPATH, selector)
                    if video_elements:
                        logger.info(f"✅ Found {len(video_elements)} videos with selector: {selector}")
                        break
                except:
                    continue
            
            if not video_elements:
                logger.error("❌ No video elements found")
                return []
            
            for i, video_element in enumerate(video_elements[:max_videos]):
                try:
                    video_info = self.extract_single_video_info(video_element, i+1)
                    if video_info:
                        videos.append(video_info)
                        logger.info(f"✅ Extracted video #{i+1}: {video_info['title'][:40]}...")
                except Exception as e:
                    logger.warning(f"⚠️ Could not extract video #{i+1}: {e}")
                    continue
            
            logger.info(f"✅ Successfully extracted {len(videos)} videos")
            return videos
            
        except Exception as e:
            logger.error(f"❌ Failed to extract videos: {e}")
            return []
    
    def extract_single_video_info(self, video_element, position):
        """Extract information from a single video element"""
        try:
            video_info = {'position': position}
            
            # Title extraction with multiple selectors
            title_selectors = [
                ".//h3//a[@id='video-title']",
                ".//a[@id='video-title']",
                ".//h3//span[@id='video-title']",
                ".//yt-formatted-string[@id='video-title']"
            ]
            
            for selector in title_selectors:
                try:
                    title_element = video_element.find_element(By.XPATH, selector)
                    title = title_element.get_attribute('title') or title_element.text
                    if title:
                        video_info['title'] = title.strip()
                        video_info['url'] = title_element.get_attribute('href')
                        break
                except:
                    continue
            
            # Channel extraction
            channel_selectors = [
                ".//ytd-channel-name//a",
                ".//a[contains(@class, 'yt-simple-endpoint')]",
                ".//yt-formatted-string[contains(@class, 'channel')]"
            ]
            
            for selector in channel_selectors:
                try:
                    channel_element = video_element.find_element(By.XPATH, selector)
                    channel = channel_element.text
                    if channel and channel != video_info.get('title', ''):
                        video_info['channel'] = channel.strip()
                        break
                except:
                    continue
            
            # Duration extraction
            try:
                duration_element = video_element.find_element(By.XPATH, ".//span[contains(@class, 'ytd-thumbnail-overlay-time-status-renderer')]")
                video_info['duration'] = duration_element.text.strip()
            except:
                video_info['duration'] = "Unknown"
            
            # Views extraction
            try:
                views_element = video_element.find_element(By.XPATH, ".//span[contains(text(), 'view')]")
                video_info['views'] = views_element.text.strip()
            except:
                video_info['views'] = "Unknown"
            
            return video_info if 'title' in video_info else None
            
        except Exception as e:
            logger.warning(f"Failed to extract video info: {e}")
            return None
    
    def filter_abyss_videos(self, videos):
        """Filter for Abyss craft specific videos"""
        try:
            abyss_keywords = [
                'abyss', 'necromancer', 'undead', 'skeleton', 'ghost',
                'shadow', 'death', 'cemetery', 'grave', 'burial',
                'lich', 'zombie', 'spirit', 'wraith'
            ]
            
            deck_keywords = [
                'deck', 'guide', 'build', 'list', 'tier', 'meta',
                'rotation', 'unlimited', 'craft', 'shadowverse'
            ]
            
            relevant_videos = []
            
            for video in videos:
                title_lower = video['title'].lower()
                abyss_score = sum(1 for keyword in abyss_keywords if keyword in title_lower)
                deck_score = sum(1 for keyword in deck_keywords if keyword in title_lower)
                
                total_score = abyss_score + deck_score
                
                # Must have at least 1 abyss keyword and 1 deck keyword
                if abyss_score >= 1 and deck_score >= 1:
                    video['relevance_score'] = total_score
                    video['abyss_score'] = abyss_score
                    video['deck_score'] = deck_score
                    relevant_videos.append(video)
            
            # Sort by relevance
            relevant_videos.sort(key=lambda x: x['relevance_score'], reverse=True)
            
            logger.info(f"✅ Found {len(relevant_videos)} Abyss craft videos")
            return relevant_videos
            
        except Exception as e:
            logger.error(f"❌ Failed to filter videos: {e}")
            return videos
    
    def create_deck_notes(self, video_info):
        """Create structured deck notes from video information"""
        try:
            deck_note = {
                'timestamp': datetime.now().strftime('%Y-%m-%d %H:%M:%S'),
                'video_title': video_info['title'],
                'channel': video_info.get('channel', 'Unknown'),
                'video_url': video_info.get('url', ''),
                'duration': video_info.get('duration', 'Unknown'),
                'views': video_info.get('views', 'Unknown'),
                'relevance_score': video_info.get('relevance_score', 0),
                'deck_analysis': self.analyze_deck_from_title(video_info['title']),
                'notes': f"Abyss Craft deck video found - {video_info['title']}"
            }
            
            self.deck_notes.append(deck_note)
            logger.info(f"✅ Created deck note for: {video_info['title'][:40]}...")
            return deck_note
            
        except Exception as e:
            logger.error(f"❌ Failed to create deck note: {e}")
            return None
    
    def analyze_deck_from_title(self, title):
        """Analyze deck type from video title"""
        try:
            title_lower = title.lower()
            
            deck_types = {
                'aggro': ['aggro', 'rush', 'fast', 'tempo'],
                'midrange': ['midrange', 'mid', 'balanced', 'value'],
                'control': ['control', 'late', 'slow', 'reactive'],
                'combo': ['combo', 'otk', 'one turn kill', 'synergy']
            }
            
            archetype_keywords = {
                'burial_rite': ['burial', 'rite', 'cemetery'],
                'lastwords': ['lastwords', 'last words', 'death'],
                'shadow': ['shadow', 'shadows'],
                'reanimate': ['reanimate', 'revive', 'resurrect'],
                'necromancy': ['necromancy', 'necromancer']
            }
            
            analysis = {
                'deck_type': 'Unknown',
                'archetype': 'Unknown',
                'keywords_found': []
            }
            
            # Determine deck type
            for deck_type, keywords in deck_types.items():
                if any(keyword in title_lower for keyword in keywords):
                    analysis['deck_type'] = deck_type
                    break
            
            # Determine archetype
            for archetype, keywords in archetype_keywords.items():
                if any(keyword in title_lower for keyword in keywords):
                    analysis['archetype'] = archetype
                    analysis['keywords_found'].extend([k for k in keywords if k in title_lower])
                    break
            
            return analysis
            
        except Exception as e:
            logger.error(f"❌ Failed to analyze deck: {e}")
            return {'deck_type': 'Unknown', 'archetype': 'Unknown', 'keywords_found': []}
    
    def save_deck_notes(self, filename=None):
        """Save deck notes to a file"""
        try:
            if not filename:
                timestamp = datetime.now().strftime('%Y%m%d_%H%M%S')
                filename = f"abyss_craft_decks_{timestamp}.json"
            
            with open(filename, 'w', encoding='utf-8') as f:
                json.dump(self.deck_notes, f, indent=2, ensure_ascii=False)
            
            logger.info(f"✅ Saved {len(self.deck_notes)} deck notes to {filename}")
            return filename
            
        except Exception as e:
            logger.error(f"❌ Failed to save deck notes: {e}")
            return None
    
    def run_abyss_search(self):
        """Complete Abyss craft deck search workflow"""
        logger.info("🌙 Starting Abyss Craft deck search...")
        
        try:
            # Step 1: Navigate to YouTube
            if not self.navigate_to_youtube():
                logger.error("❌ Failed at YouTube navigation")
                return False
            
            # Step 2: Search for Abyss craft decks
            if not self.search_abyss_craft():
                logger.error("❌ Failed at search")
                return False
            
            # Step 3: Extract video information
            all_videos = self.extract_video_info(12)
            if not all_videos:
                logger.error("❌ No videos found")
                return False
            
            # Step 4: Filter for Abyss craft videos
            abyss_videos = self.filter_abyss_videos(all_videos)
            if not abyss_videos:
                logger.warning("⚠️ No specific Abyss videos found, using all results")
                abyss_videos = all_videos[:6]
            
            # Step 5: Create deck notes
            print(f"\n🌙 Found {len(abyss_videos)} Abyss Craft Videos:")
            print("=" * 80)
            
            for i, video in enumerate(abyss_videos[:6], 1):
                deck_note = self.create_deck_notes(video)
                
                print(f"\n#{i} - {video['title']}")
                print(f"   Channel: {video.get('channel', 'Unknown')}")
                print(f"   Duration: {video.get('duration', 'Unknown')}")
                print(f"   Views: {video.get('views', 'Unknown')}")
                
                if 'relevance_score' in video:
                    print(f"   Relevance Score: {video['relevance_score']}")
                
                if deck_note and deck_note['deck_analysis']:
                    analysis = deck_note['deck_analysis']
                    print(f"   Deck Type: {analysis['deck_type']}")
                    print(f"   Archetype: {analysis['archetype']}")
            
            # Step 6: Save notes
            saved_file = self.save_deck_notes()
            if saved_file:
                print(f"\n💾 Deck notes saved to: {saved_file}")
            
            # Step 7: Display summary
            self.display_summary(abyss_videos)
            
            logger.info("🎉 Abyss Craft search completed successfully!")
            return True
            
        except Exception as e:
            logger.error(f"❌ Search workflow failed: {e}")
            return False
        
        finally:
            self.cleanup()
    
    def display_summary(self, videos):
        """Display search summary"""
        try:
            print(f"\n🌙 ABYSS CRAFT DECK SEARCH SUMMARY")
            print("=" * 50)
            print(f"Total videos found: {len(videos)}")
            print(f"Deck notes created: {len(self.deck_notes)}")
            
            # Archetype breakdown
            archetypes = {}
            deck_types = {}
            
            for note in self.deck_notes:
                analysis = note.get('deck_analysis', {})
                archetype = analysis.get('archetype', 'Unknown')
                deck_type = analysis.get('deck_type', 'Unknown')
                
                archetypes[archetype] = archetypes.get(archetype, 0) + 1
                deck_types[deck_type] = deck_types.get(deck_type, 0) + 1
            
            print(f"\n🎯 Archetype Distribution:")
            for archetype, count in archetypes.items():
                print(f"   {archetype}: {count}")
            
            print(f"\n⚔️ Deck Type Distribution:")
            for deck_type, count in deck_types.items():
                print(f"   {deck_type}: {count}")
                
        except Exception as e:
            logger.error(f"❌ Failed to display summary: {e}")
    
    def cleanup(self):
        """Close browser and cleanup"""
        try:
            if self.driver:
                time.sleep(2)
                self.driver.quit()
                logger.info("✅ Browser closed successfully")
        except Exception as e:
            logger.error(f"❌ Cleanup failed: {e}")

def main():
    """Run Abyss Craft deck finder"""
    print("🌙 Shadowverse Abyss Craft Deck Finder")
    print("=" * 50)
    print("Searching for the latest Abyss Craft deck guides...")
    
    # Run automation
    finder = ShadowverseAbyssFinder(headless=False)
    success = finder.run_abyss_search()
    
    print("\n" + "=" * 50)
    if success:
        print("🎉 ABYSS CRAFT SEARCH COMPLETED ✅")
        print("Deck notes have been created and saved!")
    else:
        print("❌ SEARCH FAILED")
        print("Check logs for details. Try running again or check internet connection.")

if __name__ == "__main__":
    main()
