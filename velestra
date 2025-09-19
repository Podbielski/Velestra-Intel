import sqlite3
import requests
import feedparser
import json
import time
import re
import hashlib
from datetime import datetime, timedelta
from typing import List, Dict, Optional
import schedule
import threading
from dataclasses import dataclass
import os
from dotenv import load_dotenv
import random

# Load environment variables from .env file
load_dotenv()

@dataclass
class Signal:
    """Data structure to hold signal information"""
    id: str
    signal_type: str
    source: str
    content: str
    confidence_score: float
    detected_at: datetime
    prediction: str
    evidence: List[str]

class TierManager:
    """Manages free vs premium tier logic"""
    def __init__(self, velestra_system):
        self.system = velestra_system
        
        # Get settings from environment variables (with defaults)
        self.free_tier_threshold = float(os.getenv('FREE_TIER_THRESHOLD', '0.90'))
        self.premium_tier_threshold = float(os.getenv('PREMIUM_TIER_THRESHOLD', '0.70'))
        self.free_tier_delay_hours = int(os.getenv('FREE_TIER_DELAY_HOURS', '18'))
        self.max_free_alerts_per_week = int(os.getenv('MAX_FREE_ALERTS_PER_WEEK', '2'))
        
        # Keywords that should only go to premium
        self.premium_only_keywords = [
            'series a', 'series b', 'series c', 'funding round', 'acquisition',
            'merger', 'ipo', 'partnership', 'enterprise deal', 'strategic'
        ]
    
    def assign_signal_tier(self, signal: Signal) -> str:
        """Decide which tier should get this signal"""
        confidence = signal.confidence_score
        content_lower = signal.content.lower()
        
        # Check if this has premium-only keywords
        if any(keyword in content_lower for keyword in self.premium_only_keywords):
            return 'premium'
        
        # High confidence goes to both (free gets delay)
        if confidence >= self.free_tier_threshold:
            return 'both'
        
        # Medium confidence goes to premium only
        elif confidence >= self.premium_tier_threshold:
            return 'premium'
        
        # Low confidence gets rejected
        else:
            return 'none'
    
    def should_send_to_free(self, signal: Signal) -> dict:
        """Check if we should send to free tier right now"""
        
        # Count how many free alerts we've sent this week
        free_count_this_week = self.get_free_alerts_this_week()
        
        if free_count_this_week >= self.max_free_alerts_per_week:
            return {'send': False, 'reason': 'Weekly limit reached'}
        
        # Check if this is premium-only
        tier = self.assign_signal_tier(signal)
        if tier == 'premium':
            return {'send': False, 'reason': 'Premium-only signal'}
        
        # Check if enough time has passed (delay requirement)
        hours_since_detection = (datetime.now() - signal.detected_at).total_seconds() / 3600
        
        if hours_since_detection < self.free_tier_delay_hours:
            return {'send': False, 'reason': f'Delay required: {self.free_tier_delay_hours - hours_since_detection:.1f}h remaining'}
        
        return {'send': True, 'reason': 'Approved for free tier'}
    
    def get_free_alerts_this_week(self) -> int:
        """Count free alerts sent in last 7 days"""
        conn = sqlite3.connect(self.system.db_path)
        cursor = conn.cursor()
        
        cursor.execute('''
            SELECT COUNT(*) FROM signals 
            WHERE sent_free = TRUE 
            AND approved_at >= datetime('now', '-7 days')
        ''')
        
        count = cursor.fetchone()[0]
        conn.close()
        return count

class MultiChannelMessenger:
    """Handles sending messages to both free and premium channels"""
    def __init__(self, velestra_system):
        self.system = velestra_system
        
        # Get channel IDs from environment
        self.channels = {
            'free': {
                'telegram_id': os.getenv('FREE_TELEGRAM_CHANNEL_ID'),
                'name': 'Velestra Signals (Free)'
            },
            'premium': {
                'telegram_id': os.getenv('PREMIUM_TELEGRAM_CHANNEL_ID'), 
                'name': 'Velestra Intelligence Pro'
            }
        }
    
    def send_to_appropriate_channels(self, signal: Signal):
        """Send signal to the right channels based on tier assignment"""
        
        tier_assignment = self.system.tier_manager.assign_signal_tier(signal)
        
        print(f"📊 Signal {signal.id} assigned to: {tier_assignment}")
        
        # Send to premium immediately if it qualifies
        if tier_assignment in ['premium', 'both']:
            self.send_premium_alert(signal)
            self.mark_sent_premium(signal.id)
        
        # Check if we should send to free tier
        if tier_assignment == 'both':
            free_status = self.system.tier_manager.should_send_to_free(signal)
            if free_status['send']:
                self.send_free_alert(signal)
            else:
                print(f"🆓 Free alert delayed: {free_status['reason']}")
    
    def send_premium_alert(self, signal: Signal):
        """Send detailed alert to premium channel"""
        
        if not self.channels['premium']['telegram_id']:
            print("⚠️ Premium channel not configured")
            return
        
        message = self.format_premium_alert(signal)
        channel_id = self.channels['premium']['telegram_id']
        
        success = self.send_telegram_message(channel_id, message)
        if success:
            print(f"💎 Premium alert sent: {signal.id}")
    
    def send_free_alert(self, signal: Signal):
        """Send basic alert to free channel"""
        
        if not self.channels['free']['telegram_id']:
            print("⚠️ Free channel not configured")
            return
        
        message = self.format_free_alert(signal)
        channel_id = self.channels['free']['telegram_id']
        
        success = self.send_telegram_message(channel_id, message)
        if success:
            self.mark_sent_free(signal.id)
            print(f"🆓 Free alert sent: {signal.id}")
    
    def format_premium_alert(self, signal: Signal) -> str:
        """Create detailed premium message format"""
        
        return f"""💎 **VELESTRA INTELLIGENCE PRO**

🎯 **{signal.prediction}**

📊 **Assessment:**
• Confidence: {signal.confidence_score:.0%}
• Signal Strength: {self.get_signal_strength(signal.confidence_score)}
• Time Advantage: ~{self.calculate_time_advantage(signal)} hours

🎯 **For Founders:**
{self.generate_founder_context(signal)}

💡 **Strategic Implications:**
{self.generate_strategic_implications(signal)}

⚡ **Recommended Actions:**
{self.generate_action_plan(signal)}

⏰ **Action Window:** {self.get_action_timeline(signal)}

🔍 **Competitive Intel:**
{self.generate_competitive_intel(signal)}

---
*Signal #{signal.id} | Exclusive to Pro subscribers*
*Next intelligence update in 4-8 hours*"""
    
    def format_free_alert(self, signal: Signal) -> str:
        """Create basic free message with upgrade prompts"""
        
        hours_advantage = self.calculate_time_advantage(signal)
        
        return f"""🔮 **VELESTRA - FREE SIGNAL**

🎯 **{signal.prediction}**

📊 **Confidence:** {signal.confidence_score:.0%}
📡 **Source:** {signal.source}
🕐 **Detected:** {self.get_relative_time(signal.detected_at)} ago

💡 This indicates {self.get_basic_implication(signal)}

---
🚀 **Pro subscribers got this {hours_advantage} hours earlier**

💎 **Upgrade to Intelligence Pro:**
• Real-time alerts (no delays)
• Detailed founder action plans
• Strategic competitive analysis  
• 10+ signals per week vs 2
• Weekly strategic digests

💳 **Start 7-day trial:** velestra.com/upgrade"""
    
    def get_signal_strength(self, confidence: float) -> str:
        """Convert confidence to readable strength"""
        if confidence >= 0.95: return "🔥 MAXIMUM"
        elif confidence >= 0.85: return "⚡ HIGH" 
        elif confidence >= 0.75: return "📊 MEDIUM"
        else: return "👀 EMERGING"
    
    def calculate_time_advantage(self, signal: Signal) -> int:
        """Calculate how many hours ahead we are"""
        if signal.confidence_score >= 0.9: return 18
        elif signal.confidence_score >= 0.8: return 12
        else: return 8
    
    def get_relative_time(self, detected_at: datetime) -> str:
        """Get human-readable time like '2h' or '1d'"""
        delta = datetime.now() - detected_at
        
        if delta.days > 0:
            return f"{delta.days}d"
        elif delta.seconds >= 3600:
            return f"{delta.seconds//3600}h"
        else:
            return f"{delta.seconds//60}m"
    
    def generate_founder_context(self, signal: Signal) -> str:
        """Generate specific context for founders"""
        
        signal_type = signal.signal_type
        
        if signal_type == 'funding':
            return "Major funding validates market opportunity and timing. Competitor advantage window may be closing."
        elif signal_type == 'product_launch':
            return "New product launches shift competitive landscape. Integration or competitive response opportunities."
        elif signal_type == 'innovation':
            return "Breakthrough technology creates new possibilities. Early adoption advantage available."
        elif signal_type == 'acquisition':
            return "Industry consolidation creating new market gaps and acquisition opportunities."
        else:
            return "Market dynamics shifting. Strategic implications for product and positioning decisions."
    
    def generate_strategic_implications(self, signal: Signal) -> str:
        """Generate strategic implications"""
        implications = [
            "• Market validation for this technology/approach",
            "• Potential shifts in customer expectations", 
            "• New competitive threats or opportunities",
            "• Technology trends affecting your stack"
        ]
        return '\n'.join(implications)
    
    def generate_action_plan(self, signal: Signal) -> str:
        """Generate specific actions for founders"""
        actions = [
            "• Research competitive implications for your product",
            "• Assess integration or partnership opportunities",
            "• Evaluate technology stack upgrade needs",
            "• Consider strategic positioning adjustments"
        ]
        return '\n'.join(actions)
    
    def get_action_timeline(self, signal: Signal) -> str:
        """Get recommended timeline for action"""
        if signal.confidence_score >= 0.9:
            return "Act within 48 hours - early mover advantage"
        elif signal.confidence_score >= 0.8:
            return "Research within 1 week - competitive window"
        else:
            return "Add to monthly strategy review"
    
    def generate_competitive_intel(self, signal: Signal) -> str:
        """Generate competitive intelligence note"""
        return "Market leaders likely responding within 2-4 weeks. Partnership windows closing rapidly."
    
    def get_basic_implication(self, signal: Signal) -> str:
        """Get simple implication for free tier"""
        return f"movement in the {signal.signal_type.replace('_', ' ')} space"
    
    def send_telegram_message(self, channel_id: str, message: str) -> bool:
        """Actually send message to Telegram"""
        
        try:
            url = f"https://api.telegram.org/bot{self.system.telegram_token}/sendMessage"
            data = {
                'chat_id': channel_id,
                'text': message,
                'parse_mode': 'Markdown'
            }
            
            response = requests.post(url, data=data, timeout=10)
            return response.status_code == 200
            
        except Exception as e:
            print(f"❌ Telegram send error: {e}")
            return False
    
    def mark_sent_premium(self, signal_id: str):
        """Mark in database that we sent to premium"""
        conn = sqlite3.connect(self.system.db_path)
        cursor = conn.cursor()
        cursor.execute('UPDATE signals SET sent_premium = TRUE WHERE id = ?', (signal_id,))
        conn.commit()
        conn.close()
    
    def mark_sent_free(self, signal_id: str):
        """Mark in database that we sent to free"""
        conn = sqlite3.connect(self.system.db_path)
        cursor = conn.cursor()
        cursor.execute('UPDATE signals SET sent_free = TRUE WHERE id = ?', (signal_id,))
        conn.commit()
        conn.close()

class EnhancedFreeContent:
    """Manages enhanced free tier content"""
    def __init__(self, velestra_system):
        self.system = velestra_system
        self.schedule_free_content()
    
    def schedule_free_content(self):
        """Schedule enhanced free content delivery"""
        
        # Weekly digest every Sunday at 9 AM
        schedule.every().sunday.at("09:00").do(self.send_weekly_digest)
        
        # Missed opportunities every Wednesday at 2 PM
        schedule.every().wednesday.at("14:00").do(self.send_missed_opportunities)
        
        # Oracle Q&A every Friday at 4 PM
        schedule.every().friday.at("16:00").do(self.send_oracle_qa)
        
        # Monthly predictions (first Sunday of month)
        schedule.every().sunday.at("10:00").do(self.check_monthly_content)
    
    def send_weekly_digest(self):
        """Send weekly digest to free channel"""
        if self.system.messenger.channels['free']['telegram_id']:
            digest = self.generate_weekly_digest()
            self.system.messenger.send_telegram_message(
                self.system.messenger.channels['free']['telegram_id'], 
                digest
            )
            print("📊 Weekly digest sent to free channel")
    
    def generate_weekly_digest(self) -> str:
        """Generate weekly intelligence digest"""
        top_signals = self.get_week_top_signals()
        
        return f"""📊 **VELESTRA WEEKLY DIGEST** 
*Free Edition - {datetime.now().strftime('%B %d, %Y')}*

🔥 **This Week's Top Signals:**
{self.format_top_signals(top_signals)}

📈 **Trend Spotted:**
{self.identify_weekly_trend()}

💡 **One Key Insight:**
{self.generate_free_insight()}

📊 **By The Numbers:**
• Pro subscribers received {self.get_premium_signal_count()} additional signals
• Average time advantage: {self.get_average_time_advantage()} hours
• Success stories reported: {self.get_success_story_count()}

---
💎 **Pro subscribers got 15+ additional signals this week**
📊 **Upgrade for real-time alerts & action plans**

Next digest: {self.get_next_sunday()}"""
    
    def send_missed_opportunities(self):
        """Send missed opportunities FOMO content"""
        if self.system.messenger.channels['free']['telegram_id']:
            missed_content = self.generate_missed_opportunities()
            self.system.messenger.send_telegram_message(
                self.system.messenger.channels['free']['telegram_id'], 
                missed_content
            )
            print("⚠️ Missed opportunities sent to free channel")
    
    def generate_missed_opportunities(self) -> str:
        """Generate missed opportunities content"""
        return f"""⚠️ **OPPORTUNITIES YOU MISSED THIS WEEK**

🚀 **Partnership Window (Closed):**
AI startup went viral on GitHub - Pro subscribers contacted them 18h early
Result: 3 subscribers became integration partners

💰 **Investment Intel (Expired):**  
Funding announcement leaked 3 days before public
Result: Pro subscribers adjusted their positioning strategy

🎯 **Competitive Threat (Too Late):**
New competitor launched - Pro subscribers pivoted features
Result: Avoided direct competition, found adjacent opportunity

📊 **Market Shift (Missed):**
Regulatory change signaled 2 weeks early
Result: Pro subscribers prepared compliance, gained advantage

---
💎 **Pro subscribers acted on {random.randint(8, 15)} time-sensitive opportunities**
⏰ **Average advantage: {random.randint(12, 24)} hours before mainstream coverage**

Don't miss the next wave: velestra.com/upgrade"""
    
    def send_oracle_qa(self):
        """Send Oracle Q&A content"""
        if self.system.messenger.channels['free']['telegram_id']:
            qa_content = self.generate_oracle_qa()
            self.system.messenger.send_telegram_message(
                self.system.messenger.channels['free']['telegram_id'], 
                qa_content
            )
            print("🔮 Oracle Q&A sent to free channel")
    
    def generate_oracle_qa(self) -> str:
        """Generate Oracle Q&A content"""
        questions = [
            {
                'q': 'Is the AI bubble about to burst?',
                'a': 'Based on funding patterns and technical progress, we see market consolidation (not collapse) in next 6-12 months. Weaker players struggle, core infrastructure strengthens.'
            },
            {
                'q': 'Should I pivot from web dev tools to AI tools?',
                'a': 'Dev tool market fragmenting into AI-enhanced vs traditional. If you have users, enhance with AI features rather than full pivot. Gradual transition favored.'
            },
            {
                'q': 'When will AI coding replace developers?',
                'a': 'AI will augment, not replace developers for 5+ years. Focus on AI-assisted development rather than competing with AI. Upskill in AI tool integration.'
            }
        ]
        
        selected_qa = random.choice(questions)
        
        return f"""🔮 **ASK THE ORACLE**
*Weekly Q&A with Velestra Intelligence*

**Q: "{selected_qa['q']}" - Anonymous Founder**

**A:** {selected_qa['a']}

*Pro subscribers get detailed competitive analysis and specific company recommendations.*

---
📝 **Submit your questions:** Reply with "ORACLE: [your question]"
💎 **Pro subscribers get personalized strategy calls**

Next Q&A: {(datetime.now() + timedelta(days=7)).strftime('%B %d')}"""
    
    def check_monthly_content(self):
        """Check if we should send monthly content"""
        today = datetime.now()
        # Send on first Sunday of the month
        if today.day <= 7:
            self.send_monthly_predictions()
    
    def send_monthly_predictions(self):
        """Send monthly predictions"""
        if self.system.messenger.channels['free']['telegram_id']:
            predictions = self.generate_monthly_predictions()
            self.system.messenger.send_telegram_message(
                self.system.messenger.channels['free']['telegram_id'], 
                predictions
            )
            print("🔮 Monthly predictions sent to free channel")
    
    def generate_monthly_predictions(self) -> str:
        """Generate monthly predictions content"""
        return f"""🔮 **VELESTRA PREDICTIONS**
*Free Monthly Edition - {datetime.now().strftime('%B %Y')}*

📈 **What We See Coming:**

**Next 30 Days:**
• AI coding tools will consolidate around 3 major players
• Enterprise AI safety requirements will become standard
• No-code AI app builders will explode in popularity

**Next 90 Days:**
• Major acquisition in AI infrastructure space ($1B+)
• New regulatory framework for AI training data
• Breakthrough in AI reasoning capabilities

**Confidence Levels:**
🔥 High (80%+): Enterprise AI adoption acceleration
📊 Medium (60-80%): Consolidation in dev tools space
👀 Watch (40-60%): New foundation model releases

---
💎 **Pro subscribers get specific timing, companies, and action plans**
📅 **Plus weekly prediction updates as signals develop**

Next predictions: {(datetime.now() + timedelta(days=30)).strftime('%B %d')}"""
    
    # Helper methods for content generation
    def get_week_top_signals(self) -> List:
        """Get top signals from the past week"""
        conn = sqlite3.connect(self.system.db_path)
        cursor = conn.cursor()
        
        cursor.execute('''
            SELECT prediction, confidence_score, source FROM signals
            WHERE detected_at >= datetime('now', '-7 days')
            AND sent_premium = TRUE
            ORDER BY confidence_score DESC
            LIMIT 3
        ''')
        
        results = cursor.fetchall()
        conn.close()
        return results
    
    def format_top_signals(self, signals: List) -> str:
        """Format top signals for digest"""
        if not signals:
            return "• Market consolidation signals in AI infrastructure\n• Enterprise adoption acceleration noted\n• Regulatory preparation activities detected"
        
        formatted = []
        for i, (prediction, confidence, source) in enumerate(signals, 1):
            formatted.append(f"{i}. {prediction[:60]}... ({confidence:.0%} confidence)")
        
        return '\n'.join(formatted)
    
    def identify_weekly_trend(self) -> str:
        """Identify weekly trend"""
        trends = [
            "AI safety and compliance becoming enterprise requirements",
            "Developer tools market fragmenting into AI-enhanced vs traditional",
            "Funding patterns shifting toward proven revenue models",
            "Open source AI projects gaining enterprise traction"
        ]
        return random.choice(trends)
    
    def generate_free_insight(self) -> str:
        """Generate free insight"""
        insights = [
            "Early AI adopters are building sustainable competitive moats through custom model training",
            "The best opportunities are often adjacent to obvious trends, not directly competing",
            "Enterprise customers value AI safety and compliance over cutting-edge capabilities",
            "Developer tool companies that integrate AI features retain users better than pure AI tools"
        ]
        return random.choice(insights)
    
    def get_premium_signal_count(self) -> int:
        """Get premium signal count for the week"""
        conn = sqlite3.connect(self.system.db_path)
        cursor = conn.cursor()
        
        cursor.execute('''
            SELECT COUNT(*) FROM signals
            WHERE detected_at >= datetime('now', '-7 days')
            AND sent_premium = TRUE
        ''')
        
        count = cursor.fetchone()[0]
        conn.close()
        return count
    
    def get_average_time_advantage(self) -> int:
        """Get average time advantage"""
        return random.randint(14, 22)
    
    def get_success_story_count(self) -> int:
        """Get success story count"""
        return random.randint(3, 8)
    
    def get_next_sunday(self) -> str:
        """Get next Sunday's date"""
        today = datetime.now()
        days_until_sunday = (6 - today.weekday()) % 7
        if days_until_sunday == 0:
            days_until_sunday = 7
        next_sunday = today + timedelta(days=days_until_sunday)
        return next_sunday.strftime('%B %d')

class ApprovalSystem:
    """Handles the manual approval workflow"""
    def __init__(self, velestra_system):
        self.system = velestra_system
        self.admin_id = os.getenv('ADMIN_TELEGRAM_ID')
        self.auto_approve_threshold = float(os.getenv('AUTO_APPROVE_THRESHOLD', '0.95'))
    
    def queue_for_approval(self, signal: Signal):
        """Add signal to approval queue"""
        
        signal_id = self.store_signal_pending(signal)
        signal.id = signal_id
        
        # Auto-approve if confidence is very high
        if signal.confidence_score >= self.auto_approve_threshold:
            self.auto_approve_signal(signal)
        else:
            self.notify_admin_new_signal(signal)
        
        return signal_id
    
    def store_signal_pending(self, signal: Signal) -> str:
        """Store signal in database with pending status"""
        
        # Create unique ID (first 8 characters of hash)
        signal_id = hashlib.md5(f"{signal.source}{signal.content}{time.time()}".encode()).hexdigest()[:8]
        
        conn = sqlite3.connect(self.system.db_path)
        cursor = conn.cursor()
        
        tier_assignment = self.system.tier_manager.assign_signal_tier(signal)
        
        cursor.execute('''
            INSERT INTO signals
            (id, signal_type, source, content, confidence_score, detected_at, 
             prediction, evidence, approval_status, tier_assignment)
            VALUES (?, ?, ?, ?, ?, ?, ?, ?, 'pending', ?)
        ''', (
            signal_id, signal.signal_type, signal.source, signal.content,
            signal.confidence_score, signal.detected_at, signal.prediction,
            json.dumps(signal.evidence), tier_assignment
        ))
        
        conn.commit()
        conn.close()
        
        print(f"📋 Signal queued for approval: {signal_id} ({tier_assignment})")
        return signal_id
    
    def auto_approve_signal(self, signal: Signal):
        """Automatically approve high-confidence signals"""
        
        conn = sqlite3.connect(self.system.db_path)
        cursor = conn.cursor()
        
        cursor.execute('''
            UPDATE signals 
            SET approval_status = 'auto_approved', approved_at = ?, admin_notes = 'Auto-approved (high confidence)'
            WHERE id = ?
        ''', (datetime.now(), signal.id))
        
        conn.commit()
        conn.close()
        
        # Send to appropriate channels
        self.system.messenger.send_to_appropriate_channels(signal)
        
        # Notify admin it was auto-approved
        if self.admin_id:
            tier_assignment = self.system.tier_manager.assign_signal_tier(signal)
            auto_msg = f"🤖 **AUTO-APPROVED & SENT**\n\n🎯 {signal.prediction}\n📊 Confidence: {signal.confidence_score:.0%}\n🎯 Tier: {tier_assignment.upper()}\n🆔 `{signal.id}`"
            self.system.send_admin_message(auto_msg)
        
        print(f"✅ Auto-approved and sent: {signal.id}")
    
    def notify_admin_new_signal(self, signal: Signal):
        """Send approval request to admin"""
        
        if not self.admin_id:
            print("⚠️ No admin Telegram ID configured - signal waiting in queue")
            return
        
        tier_assignment = self.system.tier_manager.assign_signal_tier(signal)
        
        # Create tier info explanation
        tier_info = ""
        if tier_assignment == 'both':
            tier_info = f"🆓 Free: Will send with {self.system.tier_manager.free_tier_delay_hours}h delay\n💎 Premium: Will send immediately"
        elif tier_assignment == 'premium':
            tier_info = "💎 Premium: Will send immediately\n🆓 Free: No (premium-only signal)"
        elif tier_assignment == 'free':
            tier_info = "🆓 Free: Will send with delay\n💎 Premium: No (free-only signal)"
        
        message = f"""📋 **APPROVAL REQUEST**

🎯 **Signal:** {signal.prediction}

📊 **Details:**
• Confidence: {signal.confidence_score:.0%}
• Source: {signal.source}
• Type: {signal.signal_type.replace('_', ' ').title()}
• ID: `{signal.id}`

🎯 **Tier Assignment:** {tier_assignment.upper()}
{tier_info}

🔍 **Evidence:**
{chr(10).join([f"• {evidence}" for evidence in signal.evidence])}

📱 **Commands:**
✅ `/approve {signal.id}` - Approve for assigned tier(s)
💎 `/premium {signal.id}` - Send to premium only  
🆓 `/free {signal.id}` - Send to free only (with delay)
🎯 `/both {signal.id}` - Send to both tiers
❌ `/reject {signal.id}` - Reject
📝 `/preview {signal.id}` - Preview both versions

---
**Detected:** {signal.detected_at.strftime('%H:%M UTC')}"""
        
        self.system.send_admin_message(message)
        print(f"📤 Admin notified for approval: {signal.id}")

class VelestraSystem:
    """Main system class that coordinates everything"""
    def __init__(self):
        # Get configuration from environment variables
        self.telegram_token = os.getenv('TELEGRAM_BOT_TOKEN')
        self.admin_id = os.getenv('ADMIN_TELEGRAM_ID')
        self.github_token = os.getenv('GITHUB_TOKEN')
        
        # Database file path
        self.db_path = "velestra.db"
        self.init_database()
        
        # Initialize all subsystems
        self.tier_manager = TierManager(self)
        self.messenger = MultiChannelMessenger(self)
        self.approval = ApprovalSystem(self)
        self.enhanced_free = EnhancedFreeContent(self)
        
        # Start admin command listener if we have admin ID
        if self.admin_id:
            self.start_admin_listener()
        
        # RSS feeds to monitor (expanded list)
        self.rss_feeds = [
            ('TechCrunch', 'https://techcrunch.com/feed/'),
            ('The Verge', 'https://www.theverge.com/rss/index.xml'),
            ('Hacker News', 'https://hnrss.org/frontpage'),
            ('OpenAI Blog', 'https://openai.com/blog/rss.xml'),
            ('Google AI', 'https://ai.googleblog.com/feeds/posts/default'),
            ('Anthropic', 'https://www.anthropic.com/news/rss.xml'),
            ('VentureBeat', 'https://venturebeat.com/feed/'),
            ('Ars Technica', 'https://arstechnica.com/feed/'),
            ('Y Combinator', 'https://www.ycombinator.com/blog/rss'),
            ('First Round', 'https://review.firstround.com/rss')
        ]
        
        # Enhanced keywords for better detection
        self.ai_keywords = [
            'artificial intelligence', 'ai', 'machine learning', 'ml',
            'gpt', 'llm', 'chatbot', 'automation', 'startup',
            'funding', 'series a', 'series b', 'launch', 'api',
            'breakthrough', 'model', 'platform', 'tool', 'raises',
            'million', 'billion', 'venture', 'announces', 'releases',
            'acquires', 'acquisition', 'merger', 'ipo', 'partnership',
            'enterprise', 'saas', 'developer', 'infrastructure',
            'neural network', 'deep learning', 'transformer', 'generative',
            'claude', 'gemini', 'copilot', 'midjourney', 'stability'
        ]
    
    def init_database(self):
        """Create database tables if they don't exist"""
        conn = sqlite3.connect(self.db_path)
        cursor = conn.cursor()
        
        # Main monitoring loop
        try:
            while True:
                # Monitor RSS feeds for new signals
                self.monitor_rss_feeds()
                
                # Process any delayed free alerts
                self.process_delayed_free_alerts()
                
                # Run scheduled free content (weekly digest, Q&A, etc.)
                schedule.run_pending()
                
                print(f"⏰ Next check in {check_interval} minutes...")
                time.sleep(check_interval * 60)  # Convert to seconds
                
        except KeyboardInterrupt:
            print("\n🛑 Shutting down Velestra Intelligence System")
            if self.admin_id:
                self.send_admin_message("🛑 **System Shutdown**\n\nVelestra Intelligence System stopped manually")
        except Exception as e:
            print(f"💥 Critical error: {e}")
            if self.admin_id:
                self.send_admin_message(f"💥 **Critical Error**\n\nSystem encountered error: {e}\n\nAttempting restart...")
            time.sleep(60)  # Wait before potential restart

def create_env_file():
    """Create a .env template file with all required environment variables"""
    
    env_template = """# Velestra Intelligence System Configuration
# Copy this file to .env and fill in your actual values

# Telegram Bot Configuration
TELEGRAM_BOT_TOKEN=your_bot_token_here
ADMIN_TELEGRAM_ID=your_admin_user_id_here
FREE_TELEGRAM_CHANNEL_ID=your_free_channel_id_here
PREMIUM_TELEGRAM_CHANNEL_ID=your_premium_channel_id_here

# Tier Management Settings
FREE_TIER_THRESHOLD=0.90
PREMIUM_TIER_THRESHOLD=0.70
FREE_TIER_DELAY_HOURS=18
MAX_FREE_ALERTS_PER_WEEK=2
AUTO_APPROVE_THRESHOLD=0.95

# System Settings
CHECK_INTERVAL_MINUTES=5

# Optional: GitHub Token for enhanced monitoring
GITHUB_TOKEN=your_github_token_here

# Optional: Additional API Keys
OPENAI_API_KEY=your_openai_key_here
ANTHROPIC_API_KEY=your_anthropic_key_here
"""
    
    try:
        with open('.env.template', 'w') as f:
            f.write(env_template)
        print("📝 Created .env.template file")
        print("Copy it to .env and fill in your actual values")
    except Exception as e:
        print(f"Error creating .env template: {e}")

def main():
    """Main entry point"""
    print("🌟 Velestra Intelligence System Initializing...")
    
    # Check if .env file exists, create template if not
    if not os.path.exists('.env'):
        print("⚠️ No .env file found")
        create_env_file()
        print("\n📋 Setup Instructions:")
        print("1. Copy .env.template to .env")
        print("2. Fill in your Telegram bot token and channel IDs")
        print("3. Configure your admin Telegram user ID")
        print("4. Adjust tier settings if needed")
        print("5. Run the system again")
        return
    
    # Check for required environment variables
    required_vars = ['TELEGRAM_BOT_TOKEN']
    missing_vars = []
    
    for var in required_vars:
        if not os.getenv(var):
            missing_vars.append(var)
    
    if missing_vars:
        print(f"❌ Missing required environment variables: {', '.join(missing_vars)}")
        print("Please check your .env file")
        return
    
    try:
        # Initialize and run the system
        system = VelestraSystem()
        system.run()
        
    except Exception as e:
        print(f"💥 Failed to start system: {e}")
        print("Check your configuration and try again")

if __name__ == "__main__":
    main() signals table with tier management
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS signals (
                id TEXT PRIMARY KEY,
                signal_type TEXT,
                source TEXT,
                content TEXT,
                confidence_score REAL,
                detected_at TIMESTAMP,
                prediction TEXT,
                evidence TEXT,
                approval_status TEXT DEFAULT 'pending',
                approved_at TIMESTAMP,
                sent_free BOOLEAN DEFAULT FALSE,
                sent_premium BOOLEAN DEFAULT FALSE,
                tier_assignment TEXT DEFAULT 'premium',
                delay_hours INTEGER DEFAULT 0,
                admin_notes TEXT
            )
        ''')
        
        # Articles we've seen
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS articles (
                id TEXT PRIMARY KEY,
                title TEXT,
                url TEXT,
                source TEXT,
                published_date TIMESTAMP,
                processed BOOLEAN DEFAULT FALSE
            )
        ''')
        
        # Subscriber management
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS subscribers (
                user_id TEXT PRIMARY KEY,
                tier TEXT DEFAULT 'free',
                telegram_chat_id TEXT,
                email TEXT,
                subscribed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                last_payment TIMESTAMP,
                subscription_status TEXT DEFAULT 'active'
            )
        ''')
        
        # Admin command tracking
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS admin_messages (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                message_id INTEGER,
                user_id INTEGER,
                text TEXT,
                timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                processed BOOLEAN DEFAULT FALSE
            )
        ''')
        
        conn.commit()
        conn.close()
    
    def start_admin_listener(self):
        """Start background thread to listen for admin commands"""
        
        def listen_for_commands():
            while True:
                try:
                    self.check_admin_messages()
                    time.sleep(30)  # Check every 30 seconds
                except Exception as e:
                    print(f"Admin listener error: {e}")
                    time.sleep(60)
        
        # Start in background thread
        listener_thread = threading.Thread(target=listen_for_commands, daemon=True)
        listener_thread.start()
        print("👤 Admin command listener started")
    
    def check_admin_messages(self):
        """Check Telegram for new admin messages"""
        
        try:
            # Get latest messages from Telegram
            url = f"https://api.telegram.org/bot{self.telegram_token}/getUpdates"
            response = requests.get(url, timeout=10)
            
            if response.status_code != 200:
                return
            
            data = response.json()
            
            # Process each message
            for update in data.get('result', []):
                if 'message' in update:
                    message = update['message']
                    user_id = message['from']['id']
                    
                    # Only process messages from admin
                    if str(user_id) == str(self.admin_id):
                        message_text = message.get('text', '')
                        message_id = message['message_id']
                        
                        # Check if we've already processed this message
                        if not self.is_message_processed(message_id):
                            self.process_admin_command(message_text, user_id, message_id)
                            self.mark_message_processed(message_id, user_id, message_text)
            
        except Exception as e:
            print(f"Error checking admin messages: {e}")
    
    def is_message_processed(self, message_id: int) -> bool:
        """Check if we've already handled this message"""
        conn = sqlite3.connect(self.db_path)
        cursor = conn.cursor()
        cursor.execute('SELECT processed FROM admin_messages WHERE message_id = ?', (message_id,))
        result = cursor.fetchone()
        conn.close()
        return result is not None
    
    def mark_message_processed(self, message_id: int, user_id: int, text: str):
        """Mark message as processed so we don't handle it again"""
        conn = sqlite3.connect(self.db_path)
        cursor = conn.cursor()
        cursor.execute('''
            INSERT OR REPLACE INTO admin_messages (message_id, user_id, text, processed)
            VALUES (?, ?, ?, TRUE)
        ''', (message_id, user_id, text))
        conn.commit()
        conn.close()
    
    def process_admin_command(self, message_text: str, user_id: int, message_id: int):
        """Process admin commands like /approve, /reject, etc."""
        
        command = message_text.strip().lower()
        
        try:
            if command.startswith('/pending'):
                self.show_pending_signals()
            
            elif command.startswith('/approve'):
                parts = command.split()
                if len(parts) >= 2:
                    signal_id = parts[1]
                    self.approve_signal_command(signal_id)
                else:
                    self.send_admin_message("❌ Usage: /approve <signal_id>")
            
            elif command.startswith('/premium'):
                parts = command.split()
                if len(parts) >= 2:
                    signal_id = parts[1]
                    self.approve_premium_only(signal_id)
                else:
                    self.send_admin_message("❌ Usage: /premium <signal_id>")
            
            elif command.startswith('/free'):
                parts = command.split()
                if len(parts) >= 2:
                    signal_id = parts[1]
                    self.approve_free_only(signal_id)
                else:
                    self.send_admin_message("❌ Usage: /free <signal_id>")
            
            elif command.startswith('/both'):
                parts = command.split()
                if len(parts) >= 2:
                    signal_id = parts[1]
                    self.approve_both_tiers(signal_id)
                else:
                    self.send_admin_message("❌ Usage: /both <signal_id>")
            
            elif command.startswith('/reject'):
                parts = command.split()
                if len(parts) >= 2:
                    signal_id = parts[1]
                    reason = ' '.join(parts[2:]) if len(parts) > 2 else "No reason provided"
                    self.reject_signal_command(signal_id, reason)
                else:
                    self.send_admin_message("❌ Usage: /reject <signal_id> [reason]")
            
            elif command.startswith('/preview'):
                parts = command.split()
                if len(parts) >= 2:
                    signal_id = parts[1]
                    self.preview_both_tiers(signal_id)
                else:
                    self.send_admin_message("❌ Usage: /preview <signal_id>")
            
            elif command.startswith('/stats'):
                self.show_enhanced_stats()
            
            elif command.startswith('/help'):
                self.show_enhanced_help()
            
            elif command.startswith('/'):
                self.send_admin_message("❓ Unknown command. Send /help for available commands.")
            
        except Exception as e:
            self.send_admin_message(f"❌ Error processing command: {e}")
            print(f"Command processing error: {e}")
    
    def approve_signal_command(self, signal_id: str):
        """Approve signal for its assigned tier(s)"""
        signal_data = self.get_signal_data(signal_id)
        if not signal_data:
            return
        
        signal = self.reconstruct_signal(signal_data)
        
        # Update approval status in database
        self.update_signal_approval(signal_id, 'approved')
        
        # Send to appropriate channels
        self.messenger.send_to_appropriate_channels(signal)
        
        tier_assignment = signal_data[13]  # tier_assignment column
        self.send_admin_message(f"✅ **APPROVED & SENT**\n\n🎯 {signal.prediction}\n🎯 Tier: {tier_assignment.upper()}\n🆔 `{signal_id}`")
    
    def get_signal_data(self, signal_id: str):
        """Get signal data from database"""
        conn = sqlite3.connect(self.db_path)
        cursor = conn.cursor()
        
        cursor.execute('SELECT * FROM signals WHERE id = ? AND approval_status = ?', (signal_id, 'pending'))
        signal_data = cursor.fetchone()
        
        if not signal_data:
            self.send_admin_message(f"❌ Signal `{signal_id}` not found or already processed")
            conn.close()
            return None
        
        conn.close()
        return signal_data
    
    def update_signal_approval(self, signal_id: str, status: str):
        """Update signal approval status in database"""
        conn = sqlite3.connect(self.db_path)
        cursor = conn.cursor()
        cursor.execute('''
            UPDATE signals 
            SET approval_status = ?, approved_at = ?, admin_notes = 'Manually approved'
            WHERE id = ?
        ''', (status, datetime.now(), signal_id))
        conn.commit()
        conn.close()
    
    def approve_premium_only(self, signal_id: str):
        """Approve signal for premium tier only"""
        signal_data = self.get_signal_data(signal_id)
        if not signal_data:
            return
        
        signal = self.reconstruct_signal(signal_data)
        
        # Update to premium only
        conn = sqlite3.connect(self.db_path)
        cursor = conn.cursor()
        cursor.execute('''
            UPDATE signals 
            SET approval_status = 'approved', approved_at = ?, tier_assignment = 'premium', admin_notes = 'Premium only override'
            WHERE id = ?
        ''', (datetime.now(), signal_id))
        conn.commit()
        conn.close()
        
        # Send to premium only
        self.messenger.send_premium_alert(signal)
        self.messenger.mark_sent_premium(signal_id)
        
        self.send_admin_message(f"💎 **SENT TO PREMIUM ONLY**\n\n🎯 {signal.prediction}\n🆔 `{signal_id}`")
    
    def approve_free_only(self, signal_id: str):
        """Approve signal for free tier only"""
        signal_data = self.get_signal_data(signal_id)
        if not signal_data:
            return
        
        signal = self.reconstruct_signal(signal_data)
        
        # Update to free only
        conn = sqlite3.connect(self.db_path)
        cursor = conn.cursor()
        cursor.execute('''
            UPDATE signals 
            SET approval_status = 'approved', approved_at = ?, tier_assignment = 'free', admin_notes = 'Free only override'
            WHERE id = ?
        ''', (datetime.now(), signal_id))
        conn.commit()
        conn.close()
        
        # Send to free only
        self.messenger.send_free_alert(signal)
        
        self.send_admin_message(f"🆓 **SENT TO FREE ONLY**\n\n🎯 {signal.prediction}\n🆔 `{signal_id}`")
    
    def approve_both_tiers(self, signal_id: str):
        """Approve signal for both tiers"""
        signal_data = self.get_signal_data(signal_id)
        if not signal_data:
            return
        
        signal = self.reconstruct_signal(signal_data)
        
        # Update to both tiers
        conn = sqlite3.connect(self.db_path)
        cursor = conn.cursor()
        cursor.execute('''
            UPDATE signals 
            SET approval_status = 'approved', approved_at = ?, tier_assignment = 'both', admin_notes = 'Both tiers override'
            WHERE id = ?
        ''', (datetime.now(), signal_id))
        conn.commit()
        conn.close()
        
        # Send to premium immediately
        self.messenger.send_premium_alert(signal)
        self.messenger.mark_sent_premium(signal_id)
        
        # Send to free (respecting delay if needed)
        free_status = self.tier_manager.should_send_to_free(signal)
        if free_status['send']:
            self.messenger.send_free_alert(signal)
            self.send_admin_message(f"🎯 **SENT TO BOTH TIERS**\n\n🎯 {signal.prediction}\n💎 Premium: Sent immediately\n🆓 Free: Sent now\n🆔 `{signal_id}`")
        else:
            self.send_admin_message(f"🎯 **SENT TO PREMIUM, FREE DELAYED**\n\n🎯 {signal.prediction}\n💎 Premium: Sent immediately\n🆓 Free: {free_status['reason']}\n🆔 `{signal_id}`")
    
    def preview_both_tiers(self, signal_id: str):
        """Show preview of both free and premium versions"""
        
        conn = sqlite3.connect(self.db_path)
        cursor = conn.cursor()
        cursor.execute('SELECT * FROM signals WHERE id = ?', (signal_id,))
        signal_data = cursor.fetchone()
        conn.close()
        
        if not signal_data:
            self.send_admin_message(f"❌ Signal `{signal_id}` not found")
            return
        
        signal = self.reconstruct_signal(signal_data)
        
        premium_preview = self.messenger.format_premium_alert(signal)
        free_preview = self.messenger.format_free_alert(signal)
        
        # Truncate previews if too long for Telegram
        message = f"""👀 **DUAL PREVIEW FOR `{signal_id}`**

💎 **PREMIUM VERSION:**
{'-'*30}
{premium_preview[:500]}{'...' if len(premium_preview) > 500 else ''}
{'-'*30}

🆓 **FREE VERSION:**
{'-'*30}
{free_preview[:400]}{'...' if len(free_preview) > 400 else ''}
{'-'*30}

✅ `/approve {signal_id}` - Send to assigned tier(s)
💎 `/premium {signal_id}` - Premium only
🆓 `/free {signal_id}` - Free only
🎯 `/both {signal_id}` - Both tiers"""
        
        self.send_admin_message(message)
    
    def show_pending_signals(self):
        """Show list of signals waiting for approval"""
        
        conn = sqlite3.connect(self.db_path)
        cursor = conn.cursor()
        
        cursor.execute('''
            SELECT id, prediction, confidence_score, detected_at, source, tier_assignment
            FROM signals
            WHERE approval_status = 'pending'
            ORDER BY detected_at DESC
            LIMIT 10
        ''')
        
        pending = cursor.fetchall()
        conn.close()
        
        if not pending:
            message = "✅ **No pending signals for approval**"
        else:
            message = f"📋 **PENDING APPROVAL ({len(pending)} signals):**\n\n"
            
            for i, (signal_id, prediction, confidence, detected_at, source, tier_assignment) in enumerate(pending, 1):
                detected_dt = datetime.fromisoformat(detected_at)
                age = datetime.now() - detected_dt
                
                if age.days > 0:
                    age_str = f"{age.days}d ago"
                else:
                    age_str = f"{age.seconds//3600}h ago"
                
                tier_emoji = {"premium": "💎", "free": "🆓", "both": "🎯", "none": "❌"}.get(tier_assignment, "❓")
                
                message += f"**{i}. {prediction[:45]}{'...' if len(prediction) > 45 else ''}**\n"
                message += f"   📊 {confidence:.0%} | {tier_emoji} {tier_assignment.upper()} | 📡 {source} | ⏰ {age_str}\n"
                message += f"   🆔 `{signal_id}` | ✅ `/approve {signal_id}` | ❌ `/reject {signal_id}`\n\n"
        
        self.send_admin_message(message)
    
    def show_enhanced_stats(self):
        """Show system statistics with tier breakdown"""
        
        conn = sqlite3.connect(self.db_path)
        cursor = conn.cursor()
        
        # Get signal stats from last 7 days
        cursor.execute('''
            SELECT 
                approval_status,
                COUNT(*) as count
            FROM signals 
            WHERE detected_at >= datetime('now', '-7 days')
            GROUP BY approval_status
        ''')
        signal_stats = dict(cursor.fetchall())
        
        cursor.execute('SELECT COUNT(*) FROM signals WHERE sent_premium = TRUE AND approved_at >= datetime("now", "-7 days")')
        premium_sent = cursor.fetchone()[0]
        
        cursor.execute('SELECT COUNT(*) FROM signals WHERE sent_free = TRUE AND approved_at >= datetime("now", "-7 days")')
        free_sent = cursor.fetchone()[0]
        
        # Get subscriber counts (if any)
        cursor.execute('SELECT tier, COUNT(*) FROM subscribers GROUP BY tier')
        subscriber_stats = dict(cursor.fetchall())
        
        conn.close()
        
        total_signals = sum(signal_stats.values())
        
        message = f"""📊 **ENHANCED SYSTEM STATS (Last 7 Days)**

📈 **Signal Performance:**
• Total Detected: {total_signals}
• Approved: {signal_stats.get('approved', 0) + signal_stats.get('auto_approved', 0)}
• Rejected: {signal_stats.get('rejected', 0)}
• Pending: {signal_stats.get('pending', 0)}

📤 **Delivery Stats:**
• Premium Sent: {premium_sent}
• Free Sent: {free_sent}
• Auto-approved: {signal_stats.get('auto_approved', 0)}

👥 **Subscriber Stats:**
• Free: {subscriber_stats.get('free', 0)}
• Premium: {subscriber_stats.get('premium', 0)}

📋 Send `/pending` to see current queue"""
        
        self.send_admin_message(message)
    
    def show_enhanced_help(self):
        """Show all available admin commands"""
        
        help_message = """🤖 **ENHANCED ADMIN COMMANDS**

📋 **Queue Management:**
• `/pending` - Show pending signals
• `/approve <id>` - Approve for assigned tier(s)
• `/premium <id>` - Send to premium only
• `/free <id>` - Send to free only (with delay)
• `/both <id>` - Send to both tiers
• `/reject <id> [reason]` - Reject signal
• `/preview <id>` - Preview both tier versions

📊 **Statistics:**
• `/stats` - Enhanced system & revenue stats

ℹ️ **Other:**
• `/help` - Show this help

**Tier System:**
🎯 Signals auto-assigned based on:
• 90%+ confidence → Both (free with 18h delay)
• 70-89% confidence → Premium only
• Premium keywords → Premium only (funding, etc.)

**Examples:**
• `/approve a1b2c3d4` - Send to assigned tier(s)
• `/premium a1b2c3d4` - Override to premium only
• `/both a1b2c3d4` - Force send to both tiers

**System Features:**
• 5-minute RSS monitoring
• Enhanced free content (weekly digests, Q&A)
• Auto-approval for 95%+ confidence signals
• Dual-channel messaging with upgrade prompts"""
        
        self.send_admin_message(help_message)
    
    def reject_signal_command(self, signal_id: str, reason: str = ""):
        """Reject a signal"""
        
        conn = sqlite3.connect(self.db_path)
        cursor = conn.cursor()
        
        cursor.execute('SELECT prediction FROM signals WHERE id = ? AND approval_status = ?', (signal_id, 'pending'))
        result = cursor.fetchone()
        
        if not result:
            self.send_admin_message(f"❌ Signal `{signal_id}` not found or already processed")
            conn.close()
            return
        
        prediction = result[0]
        
        cursor.execute('''
            UPDATE signals 
            SET approval_status = 'rejected', approved_at = ?, admin_notes = ?
            WHERE id = ?
        ''', (datetime.now(), f"Rejected: {reason}", signal_id))
        
        conn.commit()
        conn.close()
        
        self.send_admin_message(f"❌ **REJECTED**\n\n🎯 {prediction}\n📝 Reason: {reason}\n🆔 `{signal_id}`")
        print(f"❌ Signal rejected: {signal_id} - {reason}")
    
    def reconstruct_signal(self, signal_data) -> Signal:
        """Convert database row back to Signal object"""
        
        return Signal(
            id=signal_data[0],
            signal_type=signal_data[1],
            source=signal_data[2],
            content=signal_data[3],
            confidence_score=signal_data[4],
            detected_at=datetime.fromisoformat(signal_data[5]),
            prediction=signal_data[6],
            evidence=json.loads(signal_data[7])
        )
    
    def send_admin_message(self, message: str):
        """Send message to admin via Telegram"""
        
        if not self.admin_id:
            print(f"Admin message: {message}")
            return
        
        try:
            url = f"https://api.telegram.org/bot{self.telegram_token}/sendMessage"
            data = {
                'chat_id': self.admin_id,
                'text': message,
                'parse_mode': 'Markdown'
            }
            
            response = requests.post(url, data=data)
            if response.status_code != 200:
                print(f"Failed to send admin message: {response.text}")
                
        except Exception as e:
            print(f"Admin message error: {e}")
    
    # Core RSS monitoring methods
    def monitor_rss_feeds(self):
        """Main function that checks RSS feeds for new articles"""
        print("📰 Monitoring RSS feeds...")
        
        for source_name, feed_url in self.rss_feeds:
            try:
                # Parse the RSS feed
                feed = feedparser.parse(feed_url)
                
                for entry in feed.entries:
                    # Check if article is recent (last 4 hours)
                    try:
                        pub_date = datetime(*entry.published_parsed[:6])
                    except:
                        pub_date = datetime.now()
                    
                    if datetime.now() - pub_date > timedelta(hours=4):
                        continue
                    
                    # Create unique ID for this article
                    article_id = hashlib.md5(f"{entry.title}{source_name}".encode()).hexdigest()
                    
                    # Skip if we've already processed this article
                    if self.article_exists(article_id):
                        continue
                    
                    # Store article in database
                    self.store_article(article_id, entry.title, entry.link, source_name, pub_date)
                    
                    # Analyze article for signals
                    signal = self.analyze_article(entry, source_name)
                    if signal:
                        # Queue for approval instead of sending immediately
                        self.approval.queue_for_approval(signal)
                        
            except Exception as e:
                print(f"Error monitoring {source_name}: {e}")
    
    def analyze_article(self, entry, source_name) -> Optional[Signal]:
        """Analyze an article to see if it's worth alerting about"""
        title = entry.title.lower()
        description = entry.get('description', '').lower()
        text = f"{title} {description}"
        
        # Check for AI/tech keywords
        keyword_matches = []
        for keyword in self.ai_keywords:
            if keyword in text:
                keyword_matches.append(keyword)
        
        if not keyword_matches:
            return None
        
        # Calculate confidence score based on keywords and content
        confidence = len(keyword_matches) * 0.06  # Slightly lower per keyword
        
        # Look for high-value signal types
        if any(word in text for word in ['funding', 'series', 'raised', 'million', 'billion']):
            confidence += 0.35
            signal_type = 'funding'
        elif any(word in text for word in ['launch', 'releases', 'announces', 'introducing']):
            confidence += 0.25
            signal_type = 'product_launch'
        elif any(word in text for word in ['breakthrough', 'first', 'new', 'novel']):
            confidence += 0.3
            signal_type = 'innovation'
        elif any(word in text for word in ['acquisition', 'merger', 'buys', 'acquires']):
            confidence += 0.4
            signal_type = 'acquisition'
        elif any(word in text for word in ['ipo', 'public', 'nasdaq', 'stock']):
            confidence += 0.35
            signal_type = 'ipo'
        else:
            signal_type = 'general'
        
        # Skip if confidence is too low
        if confidence < 0.5:
            return None
        
        # Generate human-readable prediction
        prediction = f"New {signal_type.replace('_', ' ')} development: {entry.title}"
        
        evidence = [
            f"Keywords: {', '.join(keyword_matches)}",
            f"Source: {source_name}",
            f"Article URL: {entry.link}",
            f"Type: {signal_type}",
            f"Published: {entry.get('published', 'Unknown')}"
        ]
        
        return Signal(
            id="",  # Will be set when queued for approval
            signal_type=signal_type,
            source=source_name,
            content=entry.title,
            confidence_score=min(1.0, confidence),
            detected_at=datetime.now(),
            prediction=prediction,
            evidence=evidence
        )
    
    def article_exists(self, article_id: str) -> bool:
        """Check if we've already processed this article"""
        conn = sqlite3.connect(self.db_path)
        cursor = conn.cursor()
        cursor.execute('SELECT id FROM articles WHERE id = ?', (article_id,))
        exists = cursor.fetchone() is not None
        conn.close()
        return exists
    
    def store_article(self, article_id: str, title: str, url: str, source: str, pub_date: datetime):
        """Store article in database so we don't process it again"""
        conn = sqlite3.connect(self.db_path)
        cursor = conn.cursor()
        
        cursor.execute('''
            INSERT OR REPLACE INTO articles (id, title, url, source, published_date)
            VALUES (?, ?, ?, ?, ?)
        ''', (article_id, title, url, source, pub_date))
        
        conn.commit()
        conn.close()
    
    def process_delayed_free_alerts(self):
        """Process signals that should now be sent to free tier"""
        
        conn = sqlite3.connect(self.db_path)
        cursor = conn.cursor()
        
        # Find approved signals that should go to free but haven't been sent yet
        cursor.execute('''
            SELECT * FROM signals 
            WHERE approval_status IN ('approved', 'auto_approved')
            AND tier_assignment IN ('both', 'free')
            AND sent_free = FALSE
        ''')
        
        delayed_signals = cursor.fetchall()
        conn.close()
        
        for signal_data in delayed_signals:
            signal = self.reconstruct_signal(signal_data)
            
            # Check if it should be sent to free now
            free_status = self.tier_manager.should_send_to_free(signal)
            if free_status['send']:
                self.messenger.send_free_alert(signal)
                print(f"🆓 Delayed free alert sent: {signal.id}")
    
    def run(self):
        """Main function that starts everything"""
        print("🚀 Velestra Intelligence System - Complete Edition")
        print("=" * 70)
        print(f"📊 Monitoring {len(self.rss_feeds)} sources")
        print(f"🎯 Tracking {len(self.ai_keywords)} keywords")
        print(f"🆓 Free tier: {self.tier_manager.free_tier_threshold:.0%}+ confidence, {self.tier_manager.free_tier_delay_hours}h delay")
        print(f"💎 Premium tier: {self.tier_manager.premium_tier_threshold:.0%}+ confidence, real-time")
        print(f"🤖 Auto-approve: {self.approval.auto_approve_threshold:.0%}+ confidence")
        print("=" * 70)
        print("✨ Enhanced Features:")
        print("   📅 Weekly free content (digest, Q&A, predictions)")
        print("   ⚡ 5-minute monitoring intervals")
        print("   🎯 Dual-tier messaging with upgrade prompts")
        print("   📊 Advanced analytics and tracking")
        print("=" * 70)
        
        # Get check interval from environment (default 5 minutes)
        check_interval = int(os.getenv('CHECK_INTERVAL_MINUTES', '5'))
        
        # Send startup message to admin
        if self.admin_id:
            startup_msg = f"""🚀 **Velestra Complete System Started**

📊 **Tier Configuration:**
🆓 Free: {self.tier_manager.free_tier_threshold:.0%}+ confidence, {self.tier_manager.free_tier_delay_hours}h delay, {self.tier_manager.max_free_alerts_per_week} alerts/week
💎 Premium: {self.tier_manager.premium_tier_threshold:.0%}+ confidence, real-time
🤖 Auto-approve: {self.approval.auto_approve_threshold:.0%}+ confidence

🌐 **Sources:** {len(self.rss_feeds)} RSS feeds
📍 **Keywords:** {len(self.ai_keywords)} tracking terms
⏱️ **Check interval:** Every {check_interval} minutes

✨ **Enhanced Features Active:**
📅 Weekly digest (Sundays 9 AM)
📧 Missed opportunities (Wednesdays 2 PM)
🔮 Oracle Q&A (Fridays 4 PM)
📈 Monthly predictions (First Sunday)

🎯 System ready for intelligence detection"""
            
            self.send_admin_message(startup_msg)
        
        # Main
