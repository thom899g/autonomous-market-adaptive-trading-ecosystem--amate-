# Autonomous Market-Adaptive Trading Ecosystem (AMATE)

## Objective
AMATE is an advanced AI-driven trading system designed to autonomously identify and exploit profitable market opportunities. It leverages machine learning and real-time data analysis to evolve its strategies continuously, ensuring optimal performance in dynamic environments.

## Strategy
Research and implement using available tools.

## Execution Output
SUMMARY: I've architected a comprehensive Autonomous Market-Adaptive Trading Ecosystem (AMATE) with enterprise-grade components. The system includes core trading infrastructure, real-time data pipelines, ML-powered signal generation, and risk management - all designed for production deployment with Firebase for state management.

OUTPUT:
### FILE: config.py
```python
"""
AMATE Configuration Manager
Centralized configuration with environment variable management and validation
"""
import os
import json
import logging
from typing import Dict, Any, Optional
from dataclasses import dataclass, asdict
from pathlib import Path
from dotenv import load_dotenv
import firebase_admin
from firebase_admin import credentials, firestore

# Load environment variables
load_dotenv()

@dataclass
class ExchangeConfig:
    """Exchange API configuration"""
    name: str
    api_key: str = ""
    api_secret: str = ""
    testnet: bool = True
    rate_limit: int = 10
    
@dataclass
class MLConfig:
    """Machine Learning configuration"""
    model_retrain_interval_hours: int = 24
    feature_window_size: int = 100
    prediction_horizon: int = 10
    min_confidence_threshold: float = 0.65

@dataclass
class RiskConfig:
    """Risk management configuration"""
    max_position_size_usd: float = 1000.0
    max_daily_loss_percent: float = 5.0
    max_open_positions: int = 5
    stop_loss_percent: float = 2.0
    take_profit_percent: float = 3.0

class ConfigManager:
    """Centralized configuration management with validation"""
    
    def __init__(self, config_path: str = "config.json"):
        self.logger = logging.getLogger(__name__)
        self.config_path = Path(config_path)
        self._config: Dict[str, Any] = {}
        self._firebase_app = None
        
        # Initialize with defaults
        self._load_defaults()
        
        # Load from file if exists
        if self.config_path.exists():
            self._load_from_file()
        else:
            self.logger.warning(f"Config file {config_path} not found, using defaults")
            
        # Validate critical settings
        self._validate_config()
    
    def _load_defaults(self):
        """Initialize default configuration"""
        self._config = {
            "exchange": ExchangeConfig(
                name=os.getenv("EXCHANGE_NAME", "binance"),
                api_key=os.getenv("EXCHANGE_API_KEY", ""),
                api_secret=os.getenv("EXCHANGE_API_SECRET", ""),
                testnet=os.getenv("EXCHANGE_TESTNET", "true").lower() == "true"
            ),
            "ml": MLConfig(),
            "risk": RiskConfig(),
            "logging": {
                "level": os.getenv("LOG_LEVEL", "INFO"),
                "file": "amate.log",
                "max_size_mb": 100
            },
            "firebase": {
                "credential_path": os.getenv("FIREBASE_CREDENTIAL_PATH", ""),
                "project_id": os.getenv("FIREBASE_PROJECT_ID", ""),
                "database_url": os.getenv("FIREBASE_DATABASE_URL", "")
            },
            "telegram": {
                "bot_token": os.getenv("TELEGRAM_BOT_TOKEN", ""),
                "chat_id": os.getenv("TELEGRAM_CHAT_ID", "")
            }
        }
    
    def _load_from_file(self):
        """Load configuration from JSON file"""
        try:
            with open(self.config_path, 'r') as f:
                file_config = json.load(f)
                
            # Merge with defaults
            for key, value in file_config.items():
                if key in self._config:
                    if isinstance(self._config[key], dict):
                        self._config[key].update(value)
                    else:
                        self._config[key] = value
                else:
                    self._config[key] = value
                    
            self.logger.info(f"Loaded configuration from {self.config_path}")
            
        except (json.JSONDecodeError, IOError) as e:
            self.logger.error(f"Failed to load config file: {e}")
    
    def _validate_config(self):
        """Validate critical configuration parameters"""
        errors = []
        
        # Check exchange credentials if not in testnet mode
        if not self.exchange_config.testnet:
            if not self.exchange_config.api_key:
                errors.append("Exchange API key required for live trading")
            if not self.exchange_config.api_secret:
                errors.append("Exchange API secret required for live trading")
        
        # Check Firebase configuration
        if not self.firebase_config["credential_path"]:
            self.logger.warning("Firebase credential path not set, state persistence disabled")
        
        # Check Telegram configuration
        if not self.telegram_config["bot_token"]:
            self.logger.warning("Telegram bot token not configured")
        
        if errors:
            error_msg = "\n".join(errors)
            self.logger.error(f"Configuration validation failed:\n{error_msg}")
            raise ValueError("Configuration validation failed")
    
    @property
    def exchange_config(self) -> ExchangeConfig:
        return ExchangeConfig(**self._config["exchange"])
    
    @property
    def ml_config(self) -> MLConfig:
        return MLConfig(**self._config["ml"])
    
    @property
    def risk_config(self) -> RiskConfig:
        return RiskConfig(**self._config["risk"])
    
    @property
    def firebase_config(self) -> Dict[str, Any]:
        return self._config["firebase"]
    
    @property
    def telegram_config(self) -> Dict[str, Any]:
        return self._config["telegram"]
    
    def get_firebase_app(self):
        """Initialize and return Firebase app"""
        if self._firebase_app is None:
            cred_path = self.firebase_config["credential_path"]
            if cred_path and Path(cred_path).exists():
                try:
                    cred = credentials.Certificate(cred_path)
                    self._firebase_app = firebase_admin.initialize_app(
                        cred,
                        {
                            'projectId': self.firebase_config["project_id"],
                            'databaseURL': self.firebase_config["database_url"]
                        }
                    )
                    self.logger.info("Firebase app initialized successfully")
                except Exception as e:
                    self.logger.error(f"Failed to initialize Firebase: {e}")
                    raise
            else:
                self.logger.warning("Firebase credentials not found, running in local-only mode")
        
        return self._firebase_app
    
    def save(self):
        """Save current configuration to file"""
        try:
            # Convert dataclasses to dict
            config_dict = {
                "exchange": asdict(self.exchange_config),
                "ml": asdict(self.ml