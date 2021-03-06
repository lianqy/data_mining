# 数据挖掘实验报告

## 实验环境

ubuntu服务器，python环境

## 实验原理

这次实验主要是实现了Logistic Regression算法，并将其并行化后对比速度，所用数据是第二次kaggle上提供的数据，数据格式为libsvm的数据格式，且为二分类问题，即 y ∈ {0，1}。

下面简单介绍Logistic Regression的原理：

Logistic Regression主要通过计算sigmoid函数计算出属于各个类别的概率大小，对于常见的二分类问题，一般有：

![1](http://p8pbukobc.bkt.clouddn.com/1.PNG)

其中，sigmoid函数为：

![](http://p8pbukobc.bkt.clouddn.com/2.PNG)

因此我们主要的任务是求出相关的系数，常用的方法为梯度下降法。

目标函数为：

![](http://p8pbukobc.bkt.clouddn.com/3.PNG)

由于这次我们采用的数据集光训练数据就有一百多万，因此采用梯度下降时，每一次迭代都要遍历全部的样本，通过全部样本来更新权重，花费时间过长，没有选择性；若采用随机梯度下降，则是每一次采用一个样本更新的权重，这样可以使得每一次迭代的速度变快，但是不像每次迭代遍历全部样本一样往全局最优方向走，它往往能得到一个较好的结果，却不一定是全局的最优；因此这里实现采用了两者这种的带有mini-batch的梯度下降方法进行训练，通过调整超参数batch_size，我们可以自由的选择每一次迭代的样本数，当batch_size为1时为随机梯度下降，当batch_size为整个训练集大小时为批量梯度下降。

## 实验过程

### 数据的读取和预处理

由于我们数据的格式给出的时libsvm的格式，因此要对数据进行解析：

    def lordData(path):
    	data = []
    	label = []
    	with open(path, 'r') as fr:
        	for line in fr.readlines():
        	    index = line.find(' ')
        	    data.append(line[index + 1 : ])
        	    label.append(int(line[0]))
    	return data, label

	def parseData(data):
    	dataDictionary = {}
    	features = data.strip().split() #先去除换行符等，而后根据空格切片
    	for item in features:
        	index,fea = item.split(':') #解析每个index的特征
        	dataDictionary[int(index)] = fea
        	#print dataDictionary
    	return dataDictionary  #返回稀疏特征

这两个函数用于读取和解析libsvm数据格式，第一个loadData函数通过空格分开，分别读取了label和相关特征，第二个parseData函数通过对：进行切片将libsvm格式中不为0的稀疏数据存到python的字典当中，至此完成数据的读取和解析。

完成数据的读取后，我们往往也要对数据进行预处理，常见的预处理操作便是将数据标准化，通过讲数据标准化，缩小特征间的差距，使得梯度下降收敛速度更快，代码如下：

	def normalization(dataDict):
    	#对数据进行归一化
    	data_sum = sum(float(data) for i,data in dataDict.items())
    	data_sum_sq = sum(float(data)*float(data) for i,data in dataDict.items())
    	data_mean = float(data_sum / 201)  #均值
    	data_mean_sq = float(data_sum_sq / 201) #平方的均值
    	data_var = data_mean_sq - data_mean * data_mean #方差
    	data_std = math.sqrt(data_var)
    	#print data_mean
    	#print data_mean_sq
    	#print data_std
    	result = {}
    	for i in range(202):
    	    if i == 0:
    	        continue
    	    if dataDict.has_key(i):
    	        result[i] = float((float(dataDict[i]) - data_mean) / data_std)
    	    else:
    	        result[i] = float((0 - data_mean) / data_std)
    	return result

### 实现mini-batch梯度下降

这里主要给该函数设置了三个参数batch_size、learning_rate和max_iter，实现方式模仿了sklearn中常用的param参数.batch_size设置了每次迭代的数据量大小，learning_rate设置了每次学习的步长，max_iter则设置了最大迭代次数。这里batch_size的取值采取了从0开始对总数据集取值，即假如总数据集大小为100000，当我们取batch_size为1000时，经过100次迭代后会对所有数据训练一次，之后的迭代不断的重复这个过程，相关代码如下：

	def LogisticRegression(tdata, tlabel, val_data, val_label,
		param = {'learning_rate':0.01, 'batch_size':1000, 'max_iter':1000}):
    	#获取参数学习率，若无设置，默认为0.01
    	if param.has_key('learning_rate'):
        	learning_rate = param['learning_rate']
	    else:
    	    learning_rate = 0.01
    	#获取参数batchsize，若无设置，默认为1000
    	if param.has_key('batch_size'):
    	    batch = param['batch_size']
    	else:
    	    batch = 1000
    	#获取参数最大迭代数，若无设置，默认为1000
    	if param.has_key('max_iter'):
    	    max_iter = param['max_iter']
    	else:
    	    max_iter = 1000
    
    	#获取特征的系数，共有201个特征，加上常数项，因此有202个需要学习的系数
    	#这里采用随机初始化为[0,1)间的值
    	weights = [random.random() for i in range(202)]
    	totalTime = 0.0
    	#通过minibatch随机梯度下降进行系数的更新
    	for iter in range(max_iter):
    	    start = time.time()
    	    rindex = int(0 + iter * batch)
    	    while (rindex >= len(tdata)):
    	        rindex = rindex - len(tdata)
    	    error = np.zeros(201)
    	    loss = 0.0
    	    for i in range(batch):
    	        dataDict = tdata[rindex + i]
    	        if (tlabel[rindex + i] == 1):
    	            loss = loss - math.log(sigmoid(predict(weights, dataDict)))
    	        else:
    	            loss = loss - math.log(1 - sigmoid(predict(weights, dataDict))) 
    	        e_temp = sigmoid(predict(weights, dataDict)) - tlabel[rindex + i] 
    	        for index in dataDict:
    	            error[index - 1] = error[index - 1] + e_temp * dataDict[index]
    	    #计算loss
    	    loss = loss / batch
    	    print 'loss:',loss
    	    for index in dataDict:
    	        weights[index] = weights[index] - learning_rate * (float(1)/batch) * error[index - 1]
    	    end = time.time()
    	    totalTime = totalTime + float(end - start)
    	    if ((iter + 1) % 100 == 0):
    	        calAcc(weights, val_data, val_label, iter + 1)
    	print 'iter cost:', float(totalTime) / max_iter, 's in average.'
    	return weights

为了确保算法真的收敛，因此抽取了一部分数据和验证集进行了测试，得到结果如下：

![](http://p8pbukobc.bkt.clouddn.com/4.PNG)

每100次迭代对验证集进行验证，下图为第一次验证：

![](http://p8pbukobc.bkt.clouddn.com/5.PNG)

第1000次迭代验证：

![](http://p8pbukobc.bkt.clouddn.com/6.PNG)

由上可见，算法可以收敛，成功实现。

### 实现并行化计算

对速度进行的优化主要是采用了python多进程进行，由于mini-batch每次都要对batch_size大小的图片进行计算完以后再更新权重，因此想到每一次迭代将batch_size大小的图片分为若干个进程，分别计算汇总后再进行更新权重，这里主要通过python的multiprocessing实现，代码如下：

	def MultiLR(tdata, tlabel, val_data, val_label,
		param = {'learning_rate':0.01, 'batch_size':100000, 'max_iter':10, 'n_jobs':10}):
    	#获取参数学习率，若无设置，默认为0.01
    	if param.has_key('learning_rate'):
	        learning_rate = param['learning_rate']
	    else:
	        learning_rate = 0.01
	    #获取参数batchsize，若无设置，默认为100000
	    if param.has_key('batch_size'):
	        batch = param['batch_size']
	    else:
	        batch = 100000
	    #获取参数最大迭代数，若无设置，默认为10
	    if param.has_key('max_iter'):
	        max_iter = param['max_iter']
	    else:
	        max_iter = 10
	    #获取进程数，若无设置，默认为10
	    if param.has_key('n_jobs'):
	        n_jobs = param['n_jobs']
	    else:
	        n_jobs = 10
	        
	    weights = [random.random() for i in range(202)]
	    num_of_jobs = batch / n_jobs
	    totalTime = 0.0
	    for iter in range(max_iter):
	        start = time.time()
	        lock = Lock()
	        rindex = int(0 + iter * batch)
	        while (rindex >= len(tdata)):
	            rindex = rindex - len(tdata)
	        error = Array('f', np.zeros(201))
	        loss = Value('f', 0.0)
	        #开启多进程
	        processes = []
	        for i in range(0, batch, num_of_jobs):
	            if i + num_of_jobs > batch:
	                continue
	            process = Process(target=cal_loss, \
	                args=(weights, error, loss, \
						tdata[rindex+i:rindex+i+num_of_jobs],tlabel[rindex+i:rindex+i+num_of_jobs],lock))
	            processes.append(process)
	        
	        #启动多进程
	        for i in range(len(processes)):
	            processes[i].start()
	            
	        #等待多进程结束
	        for i in range(len(processes)):
	            processes[i].join()
	            #print 'Process ', i, 'ended.'
	            
	        losses = loss.value / batch
	        print 'losses:',losses
	        for index in range(201):
	            weights[index + 1] = weights[index + 1] - learning_rate * (float(1)/batch) * error[index]
	        end = time.time()
	        totalTime = totalTime + float(end - start)
	        if ((iter + 1) % 10 == 0):
	            calAcc(weights, val_data, val_label, iter + 1)
	    print 'iter cost:', float(totalTime) / max_iter, 's in average.'
	    return weights

	def cal_loss(weights, error, loss, tdata, tlabel, lock):
	    losstemp = loss.value
	    error_temp = [error[i] for i in range(201)]
	    for i in range(len(tdata)):
	        dataDict = tdata[i]
	        if (tlabel[i] == 10):
	            losstemp = losstemp - math.log(sigmoid(predict(weights, dataDict)))
	        else:
	            losstemp = losstemp - math.log(1 - sigmoid(predict(weights, dataDict))) 
	        e_temp = sigmoid(predict(weights, dataDict)) - tlabel[i] 
	        for index in dataDict:
	            error_temp[index - 1] = error_temp[index - 1] + e_temp * dataDict[index]
	    with lock:
	        loss.value += losstemp
	        for i in range(201):
	            error[i] += error_temp[i]

这里加入了cal_loss函数用来计算每个数据的梯度，最后汇总进行权重的更新，同时多设置了一个参数n_jobs用于指定开启多进程的数目。说明一下这里对batch_size的设置，由于在服务器上数据计算的很快，batch_size设置过小时，分进程反而使得速度更慢，原因在于batch_size过小时，开启多进程的代价远比单独计算来的快，因此这里设置为100000的大小，才能体现更好的体现出多进程的效果，其中这里的训练数据来自原训练数据中的[0,100000),验证数据来自原训练数据的[200000,201000),测试效果如下：

![](http://p8pbukobc.bkt.clouddn.com/7.PNG)

最终两者在验证集的精度相差不大，但是单个进程处理1次迭代即100000条数据需要25.33秒，而开启了10个进程则只需要3.6秒，速度得到了提升，可见成功实现了多进程训练逻辑回归。
