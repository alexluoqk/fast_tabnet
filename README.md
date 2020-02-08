# TabNet for fastai
> This is an adaptation of TabNet (Attention-based network for tabular data) for fastai (>=2.0) library. The original paper https://arxiv.org/pdf/1908.07442.pdf. The implementation is taken from here https://github.com/dreamquark-ai/tabnet


## Install

`pip install fast_tabnet`

## How to use

`model = TabNetModel(emb_szs, n_cont, out_sz, embed_p=0., y_range=None, 
                     n_d=8, n_a=8,
                     n_steps=3, gamma=1.3, 
                     n_independent=2, n_shared=2, epsilon=1e-15,
                     virtual_batch_size=128, momentum=0.02)`

Parameters `emb_szs, n_cont, out_sz, embed_p, y_range` are the same as for fastai TabularModel.

- n_d : int
    Dimension of the prediction  layer (usually between 4 and 64)
- n_a : int
    Dimension of the attention  layer (usually between 4 and 64)
- n_steps: int
    Number of sucessive steps in the newtork (usually betwenn 3 and 10)
- gamma : float
    Float above 1, scaling factor for attention updates (usually betwenn 1.0 to 2.0)
- momentum : float
    Float value between 0 and 1 which will be used for momentum in all batch norm
- n_independent : int
    Number of independent GLU layer in each GLU block (default 2)
- n_shared : int
    Number of independent GLU layer in each GLU block (default 2)
- epsilon: float
    Avoid log(0), this should be kept very low


## Example

Below is an example from fastai library, but the model in use is TabNet

```python
from fastai2.basics import *
from fastai2.tabular.all import *
from fast_tabnet.core import *
```

```python
path = untar_data(URLs.ADULT_SAMPLE)
df = pd.read_csv(path/'adult.csv')
df_main,df_test = df.iloc[:10000].copy(),df.iloc[10000:].copy()
df_main.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>age</th>
      <th>workclass</th>
      <th>fnlwgt</th>
      <th>education</th>
      <th>education-num</th>
      <th>marital-status</th>
      <th>occupation</th>
      <th>relationship</th>
      <th>race</th>
      <th>sex</th>
      <th>capital-gain</th>
      <th>capital-loss</th>
      <th>hours-per-week</th>
      <th>native-country</th>
      <th>salary</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>49</td>
      <td>Private</td>
      <td>101320</td>
      <td>Assoc-acdm</td>
      <td>12.0</td>
      <td>Married-civ-spouse</td>
      <td>NaN</td>
      <td>Wife</td>
      <td>White</td>
      <td>Female</td>
      <td>0</td>
      <td>1902</td>
      <td>40</td>
      <td>United-States</td>
      <td>&gt;=50k</td>
    </tr>
    <tr>
      <th>1</th>
      <td>44</td>
      <td>Private</td>
      <td>236746</td>
      <td>Masters</td>
      <td>14.0</td>
      <td>Divorced</td>
      <td>Exec-managerial</td>
      <td>Not-in-family</td>
      <td>White</td>
      <td>Male</td>
      <td>10520</td>
      <td>0</td>
      <td>45</td>
      <td>United-States</td>
      <td>&gt;=50k</td>
    </tr>
    <tr>
      <th>2</th>
      <td>38</td>
      <td>Private</td>
      <td>96185</td>
      <td>HS-grad</td>
      <td>NaN</td>
      <td>Divorced</td>
      <td>NaN</td>
      <td>Unmarried</td>
      <td>Black</td>
      <td>Female</td>
      <td>0</td>
      <td>0</td>
      <td>32</td>
      <td>United-States</td>
      <td>&lt;50k</td>
    </tr>
    <tr>
      <th>3</th>
      <td>38</td>
      <td>Self-emp-inc</td>
      <td>112847</td>
      <td>Prof-school</td>
      <td>15.0</td>
      <td>Married-civ-spouse</td>
      <td>Prof-specialty</td>
      <td>Husband</td>
      <td>Asian-Pac-Islander</td>
      <td>Male</td>
      <td>0</td>
      <td>0</td>
      <td>40</td>
      <td>United-States</td>
      <td>&gt;=50k</td>
    </tr>
    <tr>
      <th>4</th>
      <td>42</td>
      <td>Self-emp-not-inc</td>
      <td>82297</td>
      <td>7th-8th</td>
      <td>NaN</td>
      <td>Married-civ-spouse</td>
      <td>Other-service</td>
      <td>Wife</td>
      <td>Black</td>
      <td>Female</td>
      <td>0</td>
      <td>0</td>
      <td>50</td>
      <td>United-States</td>
      <td>&lt;50k</td>
    </tr>
  </tbody>
</table>
</div>



```python
cat_names = ['workclass', 'education', 'marital-status', 'occupation', 'relationship', 'race']
cont_names = ['age', 'fnlwgt', 'education-num']
procs = [Categorify, FillMissing, Normalize]
splits = RandomSplitter()(range_of(df_main))
```

```python
to = TabularPandas(df_main, procs, cat_names, cont_names, y_names="salary", splits=splits)
```

```python
dbch = to.dataloaders()
dbch.valid.show_batch()
```


<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>workclass</th>
      <th>education</th>
      <th>marital-status</th>
      <th>occupation</th>
      <th>relationship</th>
      <th>race</th>
      <th>age_na</th>
      <th>fnlwgt_na</th>
      <th>education-num_na</th>
      <th>age</th>
      <th>fnlwgt</th>
      <th>education-num</th>
      <th>salary</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>Local-gov</td>
      <td>HS-grad</td>
      <td>Married-civ-spouse</td>
      <td>Craft-repair</td>
      <td>Husband</td>
      <td>Black</td>
      <td>False</td>
      <td>False</td>
      <td>False</td>
      <td>45.000000</td>
      <td>556652.007499</td>
      <td>9.0</td>
      <td>&gt;=50k</td>
    </tr>
    <tr>
      <th>1</th>
      <td>Private</td>
      <td>Bachelors</td>
      <td>Never-married</td>
      <td>Sales</td>
      <td>Not-in-family</td>
      <td>White</td>
      <td>False</td>
      <td>False</td>
      <td>False</td>
      <td>29.000000</td>
      <td>176683.000072</td>
      <td>13.0</td>
      <td>&gt;=50k</td>
    </tr>
    <tr>
      <th>2</th>
      <td>Private</td>
      <td>Bachelors</td>
      <td>Married-civ-spouse</td>
      <td>Exec-managerial</td>
      <td>Husband</td>
      <td>White</td>
      <td>False</td>
      <td>False</td>
      <td>False</td>
      <td>29.000000</td>
      <td>194939.999936</td>
      <td>13.0</td>
      <td>&lt;50k</td>
    </tr>
    <tr>
      <th>3</th>
      <td>Private</td>
      <td>HS-grad</td>
      <td>Married-civ-spouse</td>
      <td>Transport-moving</td>
      <td>Husband</td>
      <td>White</td>
      <td>False</td>
      <td>False</td>
      <td>False</td>
      <td>29.000000</td>
      <td>52635.998841</td>
      <td>9.0</td>
      <td>&lt;50k</td>
    </tr>
    <tr>
      <th>4</th>
      <td>State-gov</td>
      <td>Some-college</td>
      <td>Married-civ-spouse</td>
      <td>Machine-op-inspct</td>
      <td>Husband</td>
      <td>White</td>
      <td>False</td>
      <td>False</td>
      <td>False</td>
      <td>49.000000</td>
      <td>122177.000557</td>
      <td>10.0</td>
      <td>&gt;=50k</td>
    </tr>
    <tr>
      <th>5</th>
      <td>Private</td>
      <td>12th</td>
      <td>Married-civ-spouse</td>
      <td>Machine-op-inspct</td>
      <td>Other-relative</td>
      <td>Other</td>
      <td>False</td>
      <td>False</td>
      <td>False</td>
      <td>28.000000</td>
      <td>158737.000048</td>
      <td>8.0</td>
      <td>&lt;50k</td>
    </tr>
    <tr>
      <th>6</th>
      <td>Private</td>
      <td>HS-grad</td>
      <td>Married-civ-spouse</td>
      <td>Machine-op-inspct</td>
      <td>Husband</td>
      <td>White</td>
      <td>False</td>
      <td>False</td>
      <td>False</td>
      <td>55.999999</td>
      <td>192868.999992</td>
      <td>9.0</td>
      <td>&gt;=50k</td>
    </tr>
    <tr>
      <th>7</th>
      <td>Self-emp-not-inc</td>
      <td>HS-grad</td>
      <td>Married-civ-spouse</td>
      <td>Craft-repair</td>
      <td>Husband</td>
      <td>White</td>
      <td>False</td>
      <td>False</td>
      <td>False</td>
      <td>56.999999</td>
      <td>65080.002276</td>
      <td>9.0</td>
      <td>&gt;=50k</td>
    </tr>
    <tr>
      <th>8</th>
      <td>Local-gov</td>
      <td>Masters</td>
      <td>Married-civ-spouse</td>
      <td>Prof-specialty</td>
      <td>Husband</td>
      <td>White</td>
      <td>False</td>
      <td>False</td>
      <td>True</td>
      <td>50.000000</td>
      <td>145165.999578</td>
      <td>10.0</td>
      <td>&gt;=50k</td>
    </tr>
    <tr>
      <th>9</th>
      <td>Private</td>
      <td>Assoc-voc</td>
      <td>Never-married</td>
      <td>Tech-support</td>
      <td>Not-in-family</td>
      <td>White</td>
      <td>False</td>
      <td>False</td>
      <td>False</td>
      <td>35.000000</td>
      <td>186034.999925</td>
      <td>11.0</td>
      <td>&lt;50k</td>
    </tr>
  </tbody>
</table>


```python
to_tst = to.new(df_test)
to_tst.process()
to_tst.all_cols.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>workclass</th>
      <th>education</th>
      <th>marital-status</th>
      <th>occupation</th>
      <th>relationship</th>
      <th>race</th>
      <th>age_na</th>
      <th>fnlwgt_na</th>
      <th>education-num_na</th>
      <th>age</th>
      <th>fnlwgt</th>
      <th>education-num</th>
      <th>salary</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>10000</th>
      <td>5</td>
      <td>10</td>
      <td>3</td>
      <td>2</td>
      <td>1</td>
      <td>2</td>
      <td>1</td>
      <td>1</td>
      <td>1</td>
      <td>0.456238</td>
      <td>1.346622</td>
      <td>1.164335</td>
      <td>0</td>
    </tr>
    <tr>
      <th>10001</th>
      <td>5</td>
      <td>12</td>
      <td>3</td>
      <td>15</td>
      <td>1</td>
      <td>4</td>
      <td>1</td>
      <td>1</td>
      <td>1</td>
      <td>-0.930752</td>
      <td>1.259253</td>
      <td>-0.419996</td>
      <td>0</td>
    </tr>
    <tr>
      <th>10002</th>
      <td>5</td>
      <td>2</td>
      <td>1</td>
      <td>9</td>
      <td>2</td>
      <td>5</td>
      <td>1</td>
      <td>1</td>
      <td>1</td>
      <td>1.040233</td>
      <td>0.152193</td>
      <td>-1.212162</td>
      <td>0</td>
    </tr>
    <tr>
      <th>10003</th>
      <td>5</td>
      <td>12</td>
      <td>7</td>
      <td>2</td>
      <td>5</td>
      <td>5</td>
      <td>1</td>
      <td>1</td>
      <td>1</td>
      <td>0.529237</td>
      <td>-0.282632</td>
      <td>-0.419996</td>
      <td>0</td>
    </tr>
    <tr>
      <th>10004</th>
      <td>6</td>
      <td>9</td>
      <td>3</td>
      <td>5</td>
      <td>1</td>
      <td>5</td>
      <td>1</td>
      <td>1</td>
      <td>1</td>
      <td>0.748235</td>
      <td>1.449564</td>
      <td>0.372169</td>
      <td>1</td>
    </tr>
  </tbody>
</table>
</div>



```python
emb_szs = get_emb_sz(to); print(emb_szs)
```

    [(10, 6), (17, 8), (8, 5), (16, 8), (7, 5), (6, 4), (2, 2), (2, 2), (3, 3)]


That's the use of the model

```python
model = TabNetModel(emb_szs, len(to.cont_names), 1, n_d=8, n_a=32, n_steps=1); 
```

```python
opt_func = partial(Adam, wd=0.01, eps=1e-5)
learn = Learner(dbch, model, MSELossFlat(), opt_func=opt_func, lr=3e-2, metrics=[accuracy])
```

```python
learn.lr_find()
```






![png](docs/images/output_18_1.png)


```python
learn.fit_one_cycle(10)
```


<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: left;">
      <th>epoch</th>
      <th>train_loss</th>
      <th>valid_loss</th>
      <th>accuracy</th>
      <th>time</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>0</td>
      <td>0.161420</td>
      <td>0.163181</td>
      <td>0.757500</td>
      <td>00:04</td>
    </tr>
    <tr>
      <td>1</td>
      <td>0.140478</td>
      <td>0.127033</td>
      <td>0.757500</td>
      <td>00:04</td>
    </tr>
    <tr>
      <td>2</td>
      <td>0.132842</td>
      <td>0.117864</td>
      <td>0.757500</td>
      <td>00:04</td>
    </tr>
    <tr>
      <td>3</td>
      <td>0.126220</td>
      <td>0.115803</td>
      <td>0.757500</td>
      <td>00:04</td>
    </tr>
    <tr>
      <td>4</td>
      <td>0.125338</td>
      <td>0.117127</td>
      <td>0.757500</td>
      <td>00:03</td>
    </tr>
    <tr>
      <td>5</td>
      <td>0.123562</td>
      <td>0.119050</td>
      <td>0.757500</td>
      <td>00:04</td>
    </tr>
    <tr>
      <td>6</td>
      <td>0.121530</td>
      <td>0.117025</td>
      <td>0.757500</td>
      <td>00:04</td>
    </tr>
    <tr>
      <td>7</td>
      <td>0.116976</td>
      <td>0.114524</td>
      <td>0.757500</td>
      <td>00:04</td>
    </tr>
    <tr>
      <td>8</td>
      <td>0.113542</td>
      <td>0.114590</td>
      <td>0.757500</td>
      <td>00:04</td>
    </tr>
    <tr>
      <td>9</td>
      <td>0.111071</td>
      <td>0.114707</td>
      <td>0.757500</td>
      <td>00:04</td>
    </tr>
  </tbody>
</table>


## Example with Bayesian Optimization

I like to tune hyperparameters for tabular models with Bayesian Optimization. You can optimize directly your metric using this approach if the metric is sensitive enough (in our example it is not and we use validation loss instead). Also, you should create the second validation set, because you will use the first as a training set for Bayesian Optimization. 


You may need to install the optimizer `pip install bayesian-optimization`

```python
from functools import lru_cache
```

```python
# The function we'll optimize
@lru_cache(1000)
def get_accuracy(n_d:Int, n_a:Int, n_steps:Int):
    model = TabNetModel(emb_szs, len(to.cont_names), 1, n_d=n_d, n_a=n_a, n_steps=n_steps);
    learn = Learner(dbch, model, MSELossFlat(), opt_func=opt_func, lr=3e-2, metrics=[accuracy])
    learn.fit_one_cycle(5)
    return -float(learn.validate(dl=learn.dls.valid)[0])
```

This implementation of Bayesian Optimization doesn't work naturally with descreet values. That's why we use wrapper with `lru_cache`.

```python
def fit_accuracy(pow_n_d, pow_n_a, pow_n_steps):
    pow_n_d, pow_n_a, pow_n_steps = map(int, (pow_n_d, pow_n_a, pow_n_steps))
    return get_accuracy(2**pow_n_d, 2**pow_n_a, 2**pow_n_steps)
```

```python
from bayes_opt import BayesianOptimization

# Bounded region of parameter space
pbounds = {'pow_n_d': (0, 8), 'pow_n_a': (0, 8), 'pow_n_steps': (0, 4)}

optimizer = BayesianOptimization(
    f=fit_accuracy,
    pbounds=pbounds,
)
```

```python
optimizer.maximize(
    init_points=15,
    n_iter=100,
)
```

```python
optimizer.max
```




    {'target': -0.11365877091884613,
     'params': {'pow_n_a': 8.0, 'pow_n_d': 1.0475135442460393, 'pow_n_steps': 0.0}}


