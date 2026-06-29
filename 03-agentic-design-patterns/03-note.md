# Pattern 2 Note: Using Pydantic for Structured Output

In Pattern 2, the notebook defines two Pydantic models:

```python
class DestinationRecommendation(BaseModel):
    destination: str
    available: bool
    best_season: str
    highlights: list[str]
    estimated_budget_usd: int


class TravelRecommendations(BaseModel):
    recommendations: list[DestinationRecommendation]
    personalized_note: str
```

`DestinationRecommendation` describes one destination. It defines the fields every destination recommendation must contain, such as the destination name, availability, best season, highlights, and estimated budget.

`TravelRecommendations` is the outer response model. It wraps a list of `DestinationRecommendation` objects and adds a `personalized_note` for the traveler.

The notebook now uses `TravelRecommendations` when creating the structured agent:

```python
structured_agent = provider.as_agent(
    name="StructuredTravelExpert",
    instructions="""You are a travel expert. Recommend destinations based on traveler preferences.
Use the get_destination_details tool.
Return exactly 3 recommendations that match the TravelRecommendations schema.""",
    tools=[get_destination_details],
    default_options={"response_format": TravelRecommendations},
)
```

The important part is:

```python
default_options={"response_format": TravelRecommendations}
```

This tells the Agent Framework to return structured data that matches the `TravelRecommendations` Pydantic model instead of plain free-form text.

After the agent runs, the validated Pydantic object is available from the response:

```python
response = await structured_agent.run(
    "Recommend 3 destinations for a culture-loving traveler with a $2500 budget"
)

travel_recommendations = response.value
```

Because `travel_recommendations` is a `TravelRecommendations` object, the notebook can safely access fields like this:

```python
travel_recommendations.personalized_note
travel_recommendations.recommendations[0].destination
```

This is the key difference from printing a normal text response. With Pydantic structured output, downstream application code can reliably consume the result without manually parsing text.