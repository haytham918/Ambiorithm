# Ambiorithm
## About
This is the spaced-repetition algorithm designed by [Haytham Tang](https://haythamtang.com) for [Ambient Learning App](https://ambient.education). This algorithm is developed based on the [SuperMemo 2 Algorithm](https://super-memory.com/english/ol/sm2.htm) with more than 120 evaluation sets of how learners interact with the cards in different scenarios. Here, I will describe how the algorithm works.
## Relative Target
*Ambiorithm* was originally designed to calculate the spaced-repetition interval for individual flashcards (fc), which was our target in the beginning. In July, we modified our schema and applied the algorithm on each learning moments (lm) instead.
## Inputs
There are three main inputs for this algorithm:  

* **`reviewRecord`**   
* **`evaluation`**  
* **`alg`**  

### **`reviewRecord`**
---   
**`reviewRecord`** refers to a particular learner's historical interaction results of the relative target (fc/lm). This contains the number of times of:  

1.  Swiped **right** to indicate **know** this target
2.  Swiped **left** to indicate **dontKnow** this target
3.  Swiped **up** to indicate they want **oneMore** card
4.  Swiped **down** to indicate a **poorCard**   

Moreover, for a multiple-choice-question card, **`reviewRecord`** also includes how many times the learner: 

1. Answered the card **correct**
2. Answered the card **incorrect** 
3. **Skipped** answering  

### **`evaluation`**:  
---
In this context, **`evaluation`** is the most recent reviewing result for the learner. There are two parts of the evaluation: the **swipeResult** and the **tapResult**. When the target is learning moment, we would only record the **`evaluation`** of the first card (i.e.: the first card displayed of a learning moment)

**swipeResult**: One of the four directions the learner swipes for the latest review of the target, which is one of **know**, **dontKnow**, **oneMore**, and **poorCard**.  

**tapResult**: Only available for a multiple-choice-question card. It is one of the three *"objective"* evaluations of the card, which could be **correct**, **incorrect**, and **skipped**.  

### **`alg`**:  
---
**`alg`** stores the scheduling state and information of the target. It contains three parts: **memFactor**, **interval**, and **nextReview**.  

**memFactor**: The floating variable stored by each learner's target used for calculation throughout.  

**interval**: The scheduling interval (days) for the target to refresh.  

**nextReview**: A number that represents the next review date for the target. The number is calculated by using a Unix timestamp divided by 86400. We update the **nextReview** variable at the end of each algorithm execution.  

## Algorithm
*Ambiorithm* relies heavily on the learner's self-evaluation and slightly on the learner's machine-evaluation (the more objective measurement for multiple-choice-question: **correct/incorrect/skipped**). It starts by checking if the target is new and initializing the necessary variables if it is.  

* If it is a new flashcard or learning moment, we would set the `memFactor` to be **1.95** and the `interval` to be **1**. The *SuperMemo 2* algorithm initializes the `memFactor` to be **2.5**, and I derived the result of **1.95** after running hundreds of datasets and believed it results in a starting factor that will schedule repetition with an excellent growth curve. `interval` of **1** means the user will get the card immediately or the next day (depends on how you add the interval):

	```
	//Initialize the memFactor, interval, and nextReview date for a new flashcard/learning moment
	double memFactor = 1.95;
	int interval = 1;
	int nextReview = Date().getTime() / 86400 + 1;
	```

* If it is not a new fc or lm, meaning the user has reviewed the card at least once, the target shall store information regarding `reviewRecord`, `evaluation`, and `alg`. We would first initialize our new `memFactor` and `interval` as the old values:  

	```
	// Assign the new memFactor and interval with the old alg values
	double memFactor = alg.memFactor;
	int interval = alg.interval;
	```
* After setting the values of the new `memFactor` and `interval`, we will calculate the `nextReview` date based on the latest `evaluation` result. We will first check if the `swipeResult` is **poorCard**, in which case we will set the `nextReview` to be **null** so the learner will not see the flashcard/learning moment again:

	```
	// Case when the learner swipes down. This target will no longer be shown. Set nextReview as null
	if(evaluation.swipeResult == "poorCard"){
		alg.nextReview = null;
	}
	```
* Otherwise, the **swipeResult** is one of **know**, **dontKnow**, and **oneMore**. We will first calculate a variable `difference` that refers to the difference between the leaner's historical record of **know** the target and the historical record of **dontKnow** the target. This variable is used in our calculations to track how well the learner knows this target during their past review sessions, which will be explained later:  

	```
	// Difference is the subtraction of how many times the learner "know" and "dontKnow" the target
	int difference = reviewRecord.know - reviewRecord.dontKnow;
	```
	
* Now, we will first consider the case when the `swipeResult` is **know**. In this case, we will increase the `memFactor` by **0.09**. This number is also decided after running all the evaluation sets. Then, we will do a condition based on the `tapResult` for a multiple-choice-question card. If the `tapResult` is **incorrect** but the learner claims they **know** the card, we will penalize the `memFactor` by a small number **0.012**. If the `tapResult` is **skipped**, we also decrease the `memFactor` by **0.01**. The logic behind this penalty is that the learner's objective machine-evaluation does not match their self-evaluation, but it could be the case that the learner misclicks or does not want to answer, so we only decrease the `memFactor` insignificantly. In addition, if the `interval` is **1** and the `difference` is greater than or equal to **3**, meaning the learner **dontKnow** the target recently but they **know** it relatively well based on the historical record, we would boost both the `interval` and the `memFactor` and directly set the `nextReview` date so the learner will not reset and review the target again repetitively:

	```
	// Case when the learner swipes right for the latest review session
	if(evaluation.swipeResult == "know"){
		// Increase the memFactor by 0.09
		memFactor += 0.09;
		
		// If it's MCQ and the learner's test evaluation is incorrect, penalize by a small number
		if(evaluation.tapResult == "incorrect"){
			memFactor -= 0.012;
		}
		
		// If it's MCQ and the learner's test evaluation is skipped, penalize by a small number
		if(evaluation.tapResult == "skipped"){
			memFactor -= 0.01;
		}
		
		/* 
		If the learner's overall reviewRecord has 3 or more "know" than "dontKnow"
		and the learner "dontKnow" after a certain point, boost the interval and the memFactor
		based on the difference so the learner is not reset scheduling
	    */
	    if(difference >= 3 && interval == 1){
	    	alg.interval = 2 + difference;
	    	alg.memFactor = Math.max(1.3, memFactor + difference * 0.12);
	    	alg.nextReview = Date().getTime() / 86400 +  alg.interval;
	    	return;
	    }
	}
	```

* When the learner swipes left indicating **dontKnow** for the `swipeResult`, we will first decrease the `memFactor` by **0.3**. Then we will check if the learner historically **know** this target relatively well based on the `difference` variable. If this is the case, we will increase the `memFactor` a little bit. We will reset the `interval` as **1** so the learner will review the target on the next date:

	```
	// Case when the learner swipes left
	else if(evaluation.swipeResult == "dontKnow"){
		// Decrease the memFactor by 0.3
		memFactor -= 0.3;
		
		/* 
		If the user dontKnow this time, but the overall record has more "know"s, we adjust the memFactor by
		a tiny number
		*/
		if(difference >= 3){
			memFactor += 0.025;
		}
		
		// Since the learner dontKnow, we would set the interval to be 1 for them to review soon
		alg.memFactor = Math.max(1.3, memFactor);
		alg.interval = 1;
		alg.nextReview = Date().getTime() / 86400 + alg.interval;
		return;
	}
	```

* The last scenario is when the `swipeResult` is **oneMore**. Under this circumstance, we will just decrease the `memFactor` by a trivial number **0.005** as the learner is likely not very confident in their knowledge retention of the target, but it could also due to the fact that the learner is studious and eager to review cards on the same topic:

	```
	// Case when the learner swipes up for the latest evaluation
	else{
		/*
		Since the learner swipes for oneMore, it's likely they are not very confident or they want to 
		review more about this concept. We decrease the memFactor only by a trivial amount 
		*/
		memFactor -= 0.005;
	}
	```
	
* In the end, if we have not set the `nextReview` directly and returned before, we will set the `memFactor`, `interval`, and `nextReview` for the target. `memFactor` will be the larger number of **1.3** and `memFactor`. The reason that `memFactor` has a bottom threshold of **1.3** is that a `memFactor` below **1.3** will result in the targets "repeated annoyingly often and always seemed to have inherent flaws in their formulation", [according to the designer of *SuperMemo 2*](https://www.supermemo.com/en/blog/application-of-a-computer-to-improve-the-results-obtained-in-working-with-the-supermemo-method).


	```
	// The new memFactor will always be at least 1.3
	alg.memFactor = Math.max(1.3, memFactor);
	
	// The new interval will be the ceiling of the old interval times the memFactor
	alg.interval = Math.ceil(interval * memFactor);
	
	// The nextReview date will be today plus the interval
	alg.nextReview = Date().getTime() / 86400 + alg.interval;
	```

*NOTE*: All the numbers in the algorithm are determined after testing with hundreds of evaluation sets.

## Contact
If you have any questions or suggestions regarding the algorithm, feel free to [email Haytham Tang](mailto: yunxuant@umich.edu).
