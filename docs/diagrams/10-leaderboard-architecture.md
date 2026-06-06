# 10 Merit-Based Leaderboard System

This diagram maps out the various components contributing to the merit-based leaderboard, showing how points from different activities aggregate to individual and club scores.

```mermaid
graph TD
    subgraph Sources
        Attendance[Event Attendance]
        Volunteer[Volunteering]
        Organizer[Organizing Events]
        Speaker[Speaking Engagements]
        Mentor[Mentoring]
        Competition[Competition Results]
    end
    
    Sources --> Calculation[Leaderboard Calculation Engine]
    
    Calculation --> Individual[Individual Leaderboard]
    Calculation --> Club[Club Leaderboard]
    
    Individual --> Rewards[Recognition & Rewards]
    Club --> Funding[Club Funding / Status]
```
