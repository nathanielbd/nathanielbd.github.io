---
title: "Hacking the Normie Test"
date: 2021-01-15
images:
tags: 
  - web scraping
  - machine learning
---

The Normie Test is a quiz with ~40 questions that places the taker on a two-axes plot relative to internet culture personality classes like 'normies' and 'simps.'

![heh normie](https://raw.githubusercontent.com/nathanielbd/normie-neural-networks/master/ex_result.png)

Notably, the quiz is not open source and it runs on server-side PHP. How is X calculated? How is Y calculated? Why am I a 'Simp' if my X-coordinate is over the line by 0.5? To answer these pressing questions, I had to reverse-engineer the Normie Test with nothing but web scraping and machine learning.

## Introduction

The Normie Test's first 20 questions are called the 'perspective questions' meant to determine the quiz taker's outlook on life. 

>Do you want to be in society?

Questions 21-34 are the 'actual' or 'normie' questions which supposedly determine whether or not the quiz taker is a 'normie.' 

>Has anyone ever described you as awkward, autistic, weird, etc.?

Questions 35-41 are 'statistical.' They ask the quiz taker their height, BMI, climate, among other things for reference.

The perspective questions have 5 answer options: Strongly Yes, Yes, Neutral, No, and Strongly No. The [change log](http://dulm.blue/normie/changelog.gimp) shows that the creator of the test, Do You Like Memes? (DULM), calls these 5-option questions 'dynamic questions.' The normie questions have some Yes-No questions and some dynamic questions. 

A few assumptions guided my investigation. Each answer option probably has a positive or negative score which affects the X- and Y-coordinates. Then the coordinates are just the sums of the scores of the quiz taker's answers. DULM writes the following subtitle for the statistical questions:

>Enter some numbers to see how normal you are.
>**WTF!!! Is this a datamine?**
>The next six questions will make your test results more accurate, as these statistics can determine the likelihood of being normal. 

From this, it sounds like DULM looks at the distribution of personality classes for different answers and somehow takes that into account for the final result. I assume they add a 'correction term' to the coordinates based on that distribution. This assumption is necessary since the [sparse statistical data](https://raw.githubusercontent.com/nathanielbd/normie-neural-networks/master/stats.json) that DULM makes public does not include any X or Y means or covariances.

## Web scraping

Due to the last assumption, the first round of web scraping answered the perspective and normie questions randomly while leaving the statistical questions in their default options (except answering male/female/other) and recorded the resulting X- and Y-coordinates. In order to solve for the weights, this first round needed to collect at least 146 samples (25 dynamic questions, 9 Yes-No questions, and the male/female/other question) for the system not to be [underdetermined](https://en.wikipedia.org/wiki/Underdetermined_system).

Also due to the last assumption, the second round of web scraping required a sample for each of the 14400 combinations (2 3-option questions, 3 4-option questions, and 2 5-option questions) of answers to the statistical questions. This time, I also recorded the resulting personality class for each sample.

During my investigation, DULM actually added their own CAPTCHA robot test, which made web scraping a little more difficult.

| ![](/captcha.png) |
|:--:|
| Shout-out to fellow Golden Gopher [Dr. Nick Hooper](https://www-users.cs.umn.edu/~hoppernj/) for co-creating CAPTCHA |

In the end, I had my [scraping script](https://github.com/nathanielbd/normie-neural-networks/blob/master/scrape.py) prompt me to answer the CAPTCHA and quit whenever the CAPTCHA expired. It would write to the same CSV file each time and would know which combination it last wrote.

It was through this web scraping that I found that the X coordinate is always in increments of 0.5 and the Y coordinate is always in increments of 0.25.

## Findings

### X- and Y- coordinates

Figuring out the weights toward the X and Y coordinates for each question option requires solving the system

$$
Ax=b
$$

where $A$ is the matrix whose rows contain the [one-hot encoded](https://en.wikipedia.org/wiki/One-hot) answers to the questions, $x$ is a vector containing the weights of the answers, and $b$ is the X or the Y coordinate. Although this can be a simple call to `numpy.linalg.solve`, I decided to solve this with linear regression. In this case, I could tell if the assumption of linear weights is wrong if there was an atrocious mean squared error.

After rounding to the nearest 0.5 for the X weights and to the nearest 0.25 for the Y weights, we obtain [this table](https://gist.github.com/nathanielbd/2969ed56fe1de2784433f83ac3a0b0ed).

#### [Normie Neural Networks](https://github.com/nathanielbd/normie-neural-networks)

Let's try doing the same thing, but with a single layer neural network. Just for fun. And also alliteration.

```python
class SingleLayerNeuralNetwork(nn.Module):
    def __init__(self, input_dim, output_dim):
        super().__init__()
        self.input_dim = input_dim
        self.output_dim = output_dim
        self.layer = nn.Linear(self.input_dim, self.output_dim, bias=False)
    
    def forward(self, data):
        out = self.layer(data)
        return out
model = SingleLayerNeuralNetwork(answers.shape[1], XY.shape[1])
optimizer = optim.SGD(model.parameters(), lr=1e-4, momentum=0.1)
criterion = nn.L1Loss()
losses = []
EPOCHS = 10000
for epoch in range(EPOCHS):
    loss = 0
    for ans, res in train_loader:
        ans, res = ans.float(), res.float()
        optimizer.zero_grad()
        pred_res = model(ans)
        train_loss = criterion(pred_res, res)
        train_loss.backward()
        optimizer.step()
        loss += train_loss.item()
    losses.append(loss)
    if (epoch+1)%100==0:
        print(f'epoch: {epoch+1}/{EPOCHS}, loss: {loss}')
    if loss <= 0.5:
        break
```

This is probably the most explainable neural net possible. It's simply optimizing

$$
\min_x \left\lVert Ax-b \right\rVert_1
$$

and the parameters of the model can actually be interpreted! They are the weights of the answers towards the X and Y coordinates.

```python
params = list(model.parameters())

x_weights = params[0][0].detach()
columns = pd.get_dummies(data[data.columns[:-2]]).columns
x_weights = pd.DataFrame(np.array([x_weights.numpy()]), columns=columns)
# observe from the .csv files that X is always in increments of 0.5
x_weights = x_weights.applymap(lambda val: round(val*2)/2)

y_weights = params[0][1].detach()
y_weights = pd.DataFrame(np.array([y_weights.numpy()]), columns=columns)
# observe from the .csv files that Y is always in increments of 0.25
y_weights = y_weights.applymap(lambda val: round(val*4)/4)

all_weights = pd.concat([x_weights, y_weights], ignore_index=True)
all_weights.index = ['X','Y']
all_weights.to_csv('weights.csv')
# same output as linear regression
```

Yes, solving a system of linear equations and linear regression are both much simpler than training a neural net. However, I just couldn't pass up making the wonderfully alliterative phrase of 'Normie Neural Networks' (NNN) a reality! Too bad I didn't do this in November...

### Statistical corrections

When we dot product the X and Y weights with our answer data, we find that the difference between the predicted and actual XY coordinates is `[-2, -0.25]`. So when the quiz taker gives the default answers to the statistical questions (besides the male/female/other one), their XY coordinates are adjusted 2 left and 0.25 down.

```python
for ans, res in data_loader:
    pred_res = torch.from_numpy(np.array([[(ans@x_tensor.T).item(),(ans@y_tensor.T).item()]]))
    diff = res.numpy().flatten()-pred_res.numpy().flatten()
    print(f'diff: {diff}')
    # [-2, -0.25]
```

Doing a similar thing to the other combinations yielded the [statistical corrections](https://github.com/nathanielbd/normie-neural-networks/blob/master/stat-corrections.csv) for every combination of answers to the statistical questions. It turns out that linear regression fails to predict the statistical corrections, so this approach may have been the way to go. However, as the Normie Test receives more and more data, the statistics may change to the point where these corrections are no longer accurate.

### Classification

The white dividing lines seen on the results page diagram are misleading. The actual class boundaries are measured in intervals of 0.5 for the X coordinates and 0.25 for the Y coordinates.

| ![Made with MSPaint!](/normie.png) |
|:--:|
| These intervals achieved 100% accuracy on the 14400 point dataset |

---

So I guess now there's a way to create an open source version of the Normie Test. And you can try to calculate the probability that your friend rates their life below 3 if you know some of their answers and their XY coordinates.
