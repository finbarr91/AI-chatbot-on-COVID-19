# AI-chatbot-on-COVID-19
An AI chatbot that was implemented using Neural network written in Python. This chatbot gives information about COVID-2019 
import tensorflow as tf

import nltk


from nltk.stem.lancaster import LancasterStemmer
stemmer = LancasterStemmer()

import numpy as np

import random
import json

import tflearn
import pickle


with open ('intents.json') as file:
    data = json.load(file)

try:
    with open('data.pickle', 'rb') as f:
        words, labels,training,output = pickle.load(f)

except:

    words = []
    labels = []
    docs_x= []
    docs_y = []

    for intent in data['intents']:
        for pattern in intent['patterns']:
            wrds = nltk.word_tokenize(pattern)
            words.extend(wrds)
            docs_x.append(wrds)
            docs_y.append(intent['tag'])

            if intent['tag'] not in labels:
                labels.append(intent['tag'])

    words = [stemmer.stem(w.lower()) for w in words if w != '?']
    words = sorted(list(set(words)))

    labels = sorted(labels)

    training = []
    output =[]

    out_empty = [0 for _ in range(len(labels))]
    for x, doc in enumerate(docs_x):
         bag =[]

         wrds = [stemmer.stem(w) for w in doc]

         for w in words:
             if w in wrds:
                 bag.append(1) # 1 means that the word exist
             else:
                 bag.append(0) # 0 means that the word doesnt exist
         output_row = out_empty[:]
         output_row[labels.index(docs_y[x])] = 1

         training.append(bag)
         output.append(output_row)

    training = np.array(training)
    output = np.array(output)

    with open('data.pickle', 'wb') as f:
        pickle.dump((words, labels, training, output), f)

tf.reset_default_graph()


net = tflearn.input_data(shape=[None, len(training[0])])
net = tflearn.fully_connected(net,8)
net = tflearn.fully_connected(net,8)
net = tflearn.fully_connected(net,len(output[0]), activation= 'softmax')
net =tflearn.regression(net)

model =tflearn.DNN(net)

try:
    model.load('model.tflearn')

except:
    model.fit(training,output, n_epoch=1000, batch_size=8, show_metric=True)
    model.save('model.tflearn')

def bag_of_words(s,words):
    bag =[0 for _ in range(len(words))]

    s_words = nltk.word_tokenize(s)
    s_words = [stemmer.stem(word.lower()) for word in s_words]

    for se in s_words:
        for i,w in enumerate(words):
            if w == se:
                bag[i] = 1
    return np.array(bag)

def chat():
    print('start talking with the bot (type quit to stop!)')
    while True:
        inp = input('You :')
        if inp.lower() == 'quit':
            break

        results = model.predict([bag_of_words(inp,words)])[0]
        results_index = np.argmax(results)
        tag = labels[results_index]
        if results[results_index]> 0.7:
            for tg in data['intents']:
                if tg['tag'] == tag:
                    responses = tg['responses']
            print(random.choice(responses))
        else:
                print("I don't understand, try again!")

chat()


