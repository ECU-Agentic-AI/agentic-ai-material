# Simple AI Chatbot using Foundry Local SDK

Below is a well documented Python example of simple chatbot using foundry local sdk.

```python

# Create virtual environmnet for foundry-local app
# - You can use whatever virtual environment tool you want
# - I recommend Anaconda/conda for Python 

# Activate the virtual environment
 
# Install required packages
# - pip install foundry-local-sdk
# - pip install openai

# Import required packages
from foundry_local_sdk import Configuration, FoundryLocalManager
from threading import Event

def main():
    
    # Events is used to send a stop signal if Application is terminated 
    cancel_event = Event()
    
    # Create foundry local configuration
    # Be cautious about specifying a ModelCacheDir.  Sharing the same model with multiple Foundry applications is 
    # valuable characteristic of Foundry Local.  If you specify a different model cache directory, then only Foundry applications 
    # that specify the same directly will use the same models.  All others will download a duplicate model.
    config = Configuration(app_name="Simple Chatbot", log_level="info")

    # Initialize FoundryLocalManager - singleton instance
    # You only have to initialize it once in your application.
    # Every where else you just need retrieve the instance.
    # Trying to call initialize a second time will throw an error.
    FoundryLocalManager.initialize(config)

    # Get instance of FoundryLocalManager
    mgr = FoundryLocalManager.instance

    # Retrieve a list of Execution Providers (EP) that are available for your system.
    # This will be different for everyone's system because Execution Providers
    # are specific the hardware, firmware, drivers, OS on your system.
    eps = mgr.discover_eps()

    for ep in eps:
        print(f"EP Name: {ep.name}")


    # Download and register EPs with Foundry.  Registration in this scenario refers to
    # Foundry Local specifically.  Registered EPs are used to identify the models available
    # to you in the catalog.  There is a more familiar/general concept of execution providers 
    # being registered, but this is referring an EP being registered with ONNX.  This only 
    # happens when ONNX actually uses an EP in an inference session.  These two registrations
    # are not related.

    # If you hover over download_and_register_eps below, you will see that it takes a list of EP names, 
    # a `lambda str, float:`, and an optional Event as a parameter. A lambda is just an anonymous method. 
    # In this case an anonymous method that takes two input parameters: a str and a float. In the documentation, 
    # it tells you that the str is the name of the EP, and the float is the percent downloaded. 
    # download_and_register_eps will periodically call your anonymous function and provide the EP name 
    # and the Percent downloaded. It is up to you what you do inside your lambda method.  

    # Retrieve list of EP names neded for download_and_register_eps
    ep_names = [ep.name for ep in eps]

    # Skips download of EP if it has previously been downloaded and new version is not available.    
    mgr.download_and_register_eps(names=ep_names, \
                                    progress_callback=lambda ep_name, percent:  \
                                        print(f"\rEP Name: {ep_name} Percent Downloaded: {percent:.2f}%", end="", flush=True), \
                                    cancel_event=cancel_event)
    print()

    # Retrieve the model catalogue
    catalog = mgr.catalog

    # Show list of available models for your system
    models = catalog.list_models()

    for m in models:
        print(m.alias)
    print()

    # Retrieve model info for a specific model that is available in the catalog
    model = catalog.get_model("qwen2.5-0.5b")

    if model is None:
        print("Model did not exist in catalog.")
        return

    # Download the model

    # download takes a lambda float: and optional cancel_event
    # for parameters.  lambda float: just means that you need to provide an
    # anonymous method with one input parameter that is a float. The documentation
    # tells us that this float represents the download progress.

    # Skips download if model has already been downloaded
    model.download(
        lambda progress: \
            print(f"\rDownloading model: {progress:.2f}%") \
    ) 
    print()
    # Load the model
    model.load()

    # Retrieve chat client from the model
    client = model.get_chat_client()

    # Create list of chat messages  - role / content
    messages = [{"role": "user", "content": "Why does the earth look blue from outer space?"}]
    
    # Complete the chat - receive inference response from LLM. 

    # Waits for the whole message to be returned
    # as opposed to streaming which responses with each token as it is predicted.    
    response = client.complete_chat(messages)

    # This is the standard shape for OpenAI chat completion response model
    # ChatCompletion
    # - List of Choice
    #   - ChatCompletionMessage
    #     - str    
    print(response.choices[0].message.content)

    # unload the model

    # When process is done, unload model to free up memory

    # This was an example of a simple chatbot, but a more sophisticated
    # chatbot or AI agent may run for an extended period of time before 
    # being shutdown.  It is important that you unload the model whenever
    # you shutdown the application.
    model.unload()


if __name__ == "__main__":
    main()

```