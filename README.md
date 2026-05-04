from genlayer import *

class GenCoinFlip(gl.Contract):
    def __init__(self):
        self.total_flips = 0
        self.heads_wins = 0
        # etc.

    @gl.public.write
    def flip(self, choice: str, bet_amount: int):
        # Use LLM for intelligent randomness + explanation
        result = self.ask_llm_validated_random(...)  # or similar pattern
        win = (result == choice)
        
        if win:
            payout = bet_amount * 2
            # transfer logic
        
        self.total_flips += 1
        if result == "heads":
            self.heads_wins += 1
        
        # emit event for frontend
