---

layout: post
categories: [Programming]
tags: [Programming]

---

One of my old post from CSDN.

	import multiprocessing, Queue
	from multiprocessing import Process
	from time import sleep
	from datetime import datetime
	
	class MultiProcessProducer(multiprocessing.Process):
		""""""
		
		#----------------------------------------------------------------------
		def __init__(self,num, queue):
			"""Constructor"""
			multiprocessing.Process.__init__(self)
			self.num = num
			self.queue = queue
          
		def run(self):
			t1 = datetime.now()
			print 'producer start', self.num, t1
			for i in range(1000000):
				self.queue.put((i, self.num))
				#print 'producer put', i, self.num
			t2 = datetime.now()
			print 'producer exit', self.num, t2
			print 'producer', self.num, t2-t1
          

	class MultiProcessConsumer(multiprocessing.Process):
		""""""
		
		#----------------------------------------------------------------------
		def __init__(self,num, queue):
			"""Constructor"""
			multiprocessing.Process.__init__(self)
			self.num = num
			self.queue = queue
          
		def run(self):
			t1 = datetime.now()
			print 'consumer start', self.num, t1
			while True:
				d = self.queue.get()
				if d != None:
					#print 'consumer get', d, self.num
					continue
				else:
					break
			t2 = datetime.now()
			print 'consumer exit', self.num, t2
			print 'consumer', self.num, t2-t1
      
	def main():
		#create queue
		queue = multiprocessing.Queue()
		
		#create processes
		producer = []
		for i in range(1):
			producer.append(MultiProcessProducer(i, queue))
          
		consumer = []
		for i in range(2):
			consumer.append(MultiProcessConsumer(i, queue))
  
		#start processes
		for i in range(len(producer)):
			producer[i].start()
          
		for i in range(len(consumer)):
			consumer[i].start()
          
		#wait for processs to exit
		for i in range(len(producer)):
			producer[i].join()
          
		for i in range(len(consumer)):
			queue.put(None)
             
		for i in range(len(consumer)):
			consumer[i].join()
          
		print 'finish'
  
    
	if __name__ == "__main__":
		main()