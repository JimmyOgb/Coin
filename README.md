# v0.2.16
# { "Depends": "py-genlayer:1jb45aa8ynh2a9c9xn3b7qqh8sm5q93hwfp7jqmwsfhh8jpz09h6" }
from genlayer import *
import json
import typing

class GenFlip(gl.Contract):
    total_flips: u256
    heads_wins: u256
    biggest_win: u256

    def init(self):
        self.total_flips = u256(0)
        self.heads_wins = u256(0)
        self.biggest_win = u256(0)

    @gl.public.write.payable
    def flip(self, choice: str) -> dict:
        choice = choice.lower()
        if choice not in ["heads", "tails"]:
            raise gl.vm.UserError("Choice must be heads or tails")

        bet = gl.message.value
        if bet == u256(0):
            raise gl.vm.UserError("Send GLT to bet")

        # 1. We move the prompt logic out to ensure it's self-contained
        # 2. We include gl.message.hash to give the LLM a deterministic 'seed'
        def get_random_result() -> str:
            prompt = f"""
            You are a fair coin flipper. 
            The unique transaction ID is {gl.message.hash}.
            Based on this ID, flip a coin.
            Return ONLY a JSON object: {{"result": "heads"}} or {{"result": "tails"}}
            """
            return gl.nondet.exec_prompt(prompt)

        # Consensus happens here
        raw_response = gl.eq_principle.strict_eq(get_random_result)
        
        # Robust parsing to handle LLM quirks
        try:
            # Strip markdown and whitespace
            cleaned = raw_response.strip().replace("```json", "").replace("```", "")
            result_data = json.loads(cleaned)
            result = str(result_data["result"]).lower()
        except Exception:
            # Fallback if LLM output is malformed
            raise gl.vm.UserError("Consensus failed: LLM output was unreadable")

        self.total_flips += u256(1)
        win = (result == choice)
        payout = bet * u256(2) if win else u256(0)

        if win:
            if result == "heads":
                self.heads_wins += u256(1)
            if payout > self.biggest_win:
                self.biggest_win = payout
            
            # Send the payout to the gambler
            gl.send(gl.message.sender_address, payout)

        return {
            "result": result,
            "win": win,
            "payout": str(payout),
            "total_flips": str(self.total_flips)
        }

    @gl.public.view
    def get_stats(self) -> dict:
        # Avoid float division for better determinism in contract environments
        # Calculate win rate as a basis point (e.g., 5000 = 50.00%)
        win_rate_bps = 0
        if self.total_flips > u256(0):
            win_rate_bps = (self.heads_wins * u256(10000)) // self.total_flips
            
        return {
            "total_flips": str(self.total_flips),
            "heads_win_rate_bps": str(win_rate_bps),
            "biggest_win": str(self.biggest_win)
        }
