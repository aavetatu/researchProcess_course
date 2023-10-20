# Instructions on how I tested the speed of brute-forcing passwords with Hashcat and how to replicate the research

This research was made on Debian 11. Hardware included NVIDIA GeForce RTX 3060 Ti GPU, Ryzen 5 3600 CPU and Samsung 860 QVO SSD. This hardware has been in use for multiple years and it might influence the test results.

### Installing prerequisites

First I made a fresh installation of Debian 11 on my spare SSD. I considered using Kali Linux or Debian 11 on a virtual machine but decided the results would be more or at least as accurate if done with a real computer.
Instuctions for installing Debian 11 on a virtual machine or on hardware can be found From Tero Karvinen's [blog](https://terokarvinen.com/2021/install-debian-on-virtualbox/?fromSearch=debian).

After operating system installation was ready, I installed a few tools to make the research possible and my job easier. I installed [Hashcat](https://hashcat.net/hashcat/) which is self-proclaimed the fastest password cracking tool out there and Micro which is my preferred text editor.

	$ sudo apt-get update
	$ sudo apt-get -y install hashcat micro
	
I used instructions found from [Debian's wiki](https://wiki.debian.org/NvidiaGraphicsDrivers) and installed NVIDIA graphics drivers.

I also needed to install CUDA computing platform on Linux to be able to use GPU for password cracking. I followed two installation guides made by NVIDIA. The installation guides can be found from [here](https://docs.nvidia.com/cuda/cuda-installation-guide-linux/index.html#debian)
and [here](https://developer.nvidia.com/cuda-downloads?target_os=Linux&target_arch=x86_64&Distribution=Debian&target_version=11&target_type=deb_local).

Commands I used to install CUDA computing platform are listed below and are ONLY a reference for me to use if I ever need to install CUDA again. There might be some commands that are unnecessary and possibly harmful on your system. Explanations for these commands and instrctions can be found from the official CUDA installation guides.

	$ lspci | grep -i nvidia
	$ wget https://developer.download.nvidia.com/compute/cuda/12.3.0/local_installers/cuda-repo-debian11-12-3-local_12.3.0-545.23.06-1_amd64.deb
	$ sudo dpkg -i cuda-repo-debian11-12-3-local_12.3.0-545.23.06-1_amd64.deb
	$ sudo cp /var/cuda-repo-debian11-12-3-local/cuda-*-keyring.gpg /usr/share/keyrings/
	$ sudo add-apt-repository contrib
	$ sudo apt-get update
	$ sudo apt-get -y install cuda-toolkit-12-3
	$ sudo apt-get install -y cuda-drivers
	$ sudo apt-get install nvidia-libopencl1 ocl-icd-libopencl1 
	$ sudo apt-get upgrade
	$ sudo apt-get -y install cuda-toolkit-12-3
	$ sudo apt-get install -y cuda-drivers
	$ uname -m && cat /etc/*release
	$ gcc --version
	$ uname -r
	$ sudo apt-get install linux-headers-5.10.0-16-amd64
	$ sudo add-apt-repository contrib
	$ wget https://developer.download.nvidia.com/compute/cuda/repos/debian11/x86_64/cuda-keyring_1.1-1_all.deb
	$ sudo dpkg -i cuda-keyring_1.1-1_all.deb
	$ sudo apt-get update
	$ sudo apt-get -y install cuda
	$ sudo reboot


### Creating a dictionary for the attack

Next step was to make a new directory where I can store every file needed for simulating the attack

	$ cd
	$ mkdir research
	$ cd research/

`Cd` is used to move between directories and by default it goes to your home directory in Linux. Your home directory is basically the same as "C:\Users\user\" on Windows. `Mkdir` makes a new directory. With `cd research/` I moved to directory called "research".

I made a new python file "generatePasswordList.py" which I could use to generate a list with every combination of six lowercase alphabets from a-z. I used the code found from [Stackoverflow](https://stackoverflow.com/questions/45990454/generating-all-possible-combinations-of-characters-in-a-string).

	$ micro generatePasswordList.py
	$ cat generatePasswordList.py 
	import itertools
	
	def foo(i):
	    yield from itertools.product(*([i] * 6))
	
	for x in foo('abcdefghijklmnopqrstuvwxyz'):
	    print(''.join(x))

I chose to use combinations of only six lowercase letters because that filled the requirements to Facebook's account registration and making files with all possible combinations of predefined letters takes a lot of space on a hard drive and this was the easiest method to get reliable results.
Also results with more complicated passwords and password policies can be easily calculated very accurately since the speed of brute-forcing is nearly the same in longer periods of time. Time to crack more complicated or longer passwords can be predicted by comparing the amount of
all possible combinations for a password fulfilling requirements in the first password policy to the amount of all possible combinations for a password following the rules of the second password policy.

	$ time python3 generatePasswordList.py > passwordList.txt
	
	real	1m44,425s
	user	1m43,241s
	sys	0m1,096s

The command above ran generatePasswordList.py and stored the results - every combination of six lowercase letter combinations - to a file named passwordList.txt. The script took one minute and 44 seconds to run.

In Facebook's case password must contain at least six characters. The amount of six lowercase letters (a-z) is 26x26x26x26x26x26 = 26^6 = 308,915,776. This number should be the same as the number of different "words" in passwordList.txt and can be confirmed by using 'wc' which stands for word count.

	$ wc -l passwordList.txt 
	308915776 passwordList.txt
	
There are more than 300 million different combinations of six lowercase letters and generating that list took almost two minutes, generating all possible seven character combinations would take 26 times longer and the file would also be 26 times longer and bigger.
Brute-forcing a seven-character long password would also take 26 times longer. Math makes testing different password policy cases unnecessary because of how time-consuming cracking passwords can be and how much resources on a computer it uses.

### Creating a "database" to store hashed passwords

Next step was to come up with different six letter strings which represent possible passwords for Facebook users. I used online [password generator](https://passwordsgenerator.net/) where I defined passwords to contain exactly six characters and every character should be a lower-cased letter.

![image](https://github.com/aavetatu/tutkimusprosessi_raportti/assets/52470440/a9a537fc-5d99-4246-a38a-ff215788f58f)

Usually passwords are stored as hashes in a database, and they can't be reversed to be plain text. Password cracking is done by hashing potential passwords and comparing the generated hash to every hash in the database, if a match is found the string that generated said hash is the password.
This is also done while logging in to different services. Your input in the password field is ran through a hashing algorithm and if it matches the hash stored with the account in the database you are trying to log in to, you will be logged in.

I created a database (a text file on my computer) which stored the generated passwords hashed with md5 hashing algorithm. To come up with this command I found a helpful thread from [AskUbuntu](https://askubuntu.com/questions/53846/how-to-get-the-md5-hash-of-a-string-directly-in-the-terminal).

	$ printf '%s' "dfppck" | md5sum | cut -d ' ' -f 1 | tee >> hashes.txt.ntds
	$ printf '%s' "tyzepx" | md5sum | cut -d ' ' -f 1 | tee >> hashes.txt.ntds
	$ printf '%s' "fznvtp" | md5sum | cut -d ' ' -f 1 | tee >> hashes.txt.ntds
	$ printf '%s' "phqrrf" | md5sum | cut -d ' ' -f 1 | tee >> hashes.txt.ntds
	$ printf '%s' "zhixln" | md5sum | cut -d ' ' -f 1 | tee >> hashes.txt.ntds

After creating the database called "hashes.txt.ntds" I printed the contents (hashes) to the terminal with 'cat'. The hashes are in the same order as the passwords have been hashed, one password or hash per row (dfppck is the first has, zhixln is the fifth hash).

	$ cat hashes.txt.ntds 
	e6a85b22dfd21b9c0eed5591abf470bf
	3a80eb627647123cb8c5036652ac5923
	c11d2d77801b790fc1cf62b9d9833df6
	d82f08920c21a0881c1e8fd2663b3fd3
	a9c1e5c83902588a1e3b3f0caff77c63


### Brute-force attack
	
The passwords in the database were ready to be cracked with Hashcat. To be exact, the attack I used is called a dictionary attack but because the word list used contains every combination of six lowercase letters, it does the same job as an actual brute-force attack would.
At the time of simulating the attack, I couldn't figure out how to use the brute-force attack-mode with Hashcat. I used Tero Karvinen's [article](https://terokarvinen.com/2022/cracking-passwords-with-hashcat/) and Infinite Logins' YouTube [video](https://www.youtube.com/watch?v=Ndm5t4sy8o0) as refrences on how to use Hashcat.

	$ hashcat -m 0 hashes.txt.ntds passwordList.txt -o solved.txt
	...
	Started: Fri Oct 20 03:28:43 2023
	Stopped: Fri Oct 20 03:28:59 2023

`Hashcat` is used to start the program, `-m 0` is the mode it tries to crack the passwords with, in this case it is md5, `hashes.txt.ntds` is the database, `passwordList.txt` is the dictionary where Hashcat gets the strings that will be hashed and compared to the database, `-o solved.txt` outputs the cracked passwords to a text file called "solved.txt" in the order they are cracked.
The other rows are taken from the bottom of the output of hashcat and they are the only other necessary rows for this research because they represent how long it took to crack those passwords, 16 seconds. 

I used `cat` to print the contents of "solved.txt" to the terminal and from the output we can see they are cracked in the alphabetical order. This made me wonder how much the results would lie because Hashcat didn't actually need to go through all the combinations before it cracked all the passwords in my database.

	$ cat solved.txt 
	e6a85b22dfd21b9c0eed5591abf470bf:dfppck
	c11d2d77801b790fc1cf62b9d9833df6:fznvtp
	d82f08920c21a0881c1e8fd2663b3fd3:phqrrf
	3a80eb627647123cb8c5036652ac5923:tyzepx
	a9c1e5c83902588a1e3b3f0caff77c63:zhixln

I decided to create hashes for "aaaaaa" and "zzzzzz" because they are the first and the last possible password with the requirements used in this research in alphabetical order.


### Testing the accuracy of the results

This time I didn't store the hashes in a file but printed them to the terminal and used the printed hashes themselves instead of a database file in Hashcat to be cracked.

	$ printf '%s' "aaaaaa" | md5sum | cut -d ' ' -f 1
	0b4e7a0e5fe84ad35fb5f95b9ceeac79

	$ hashcat -m 0 '0b4e7a0e5fe84ad35fb5f95b9ceeac79' passwordList.txt -o solved.txt
	...
	Started: Fri Oct 20 03:33:11 2023
	Stopped: Fri Oct 20 03:33:12 2023

Cracking "aaaaaa" was instantaneous even though the output says it took a second. This was expected because it was the first string in "passwordList.txt".

	$ printf '%s' "zzzzzz" | md5sum | cut -d ' ' -f 1
	453e41d218e071ccfb2d1c99ce23906a

	$ hashcat -m 0 '453e41d218e071ccfb2d1c99ce23906a' passwordList.txt -o solved.txt
	...
	Started: Fri Oct 20 03:33:31 2023
	Stopped: Fri Oct 20 03:33:49 2023

Cracking "zzzzzz" took 18 seconds and in that time Hashcat tried oves 308 million different passwords before it cracked it. Hashcat tried 308,915,776 different passwords in 18 (or under) which equals to about 17 million passwords every second.


## Sources: 

https://askubuntu.com/questions/53846/how-to-get-the-md5-hash-of-a-string-directly-in-the-terminal

https://developer.nvidia.com/cuda-downloads?target_os=Linux&target_arch=x86_64&Distribution=Debian&target_version=11&target_type=deb_local

https://docs.nvidia.com/cuda/cuda-installation-guide-linux/index.html#debian

https://hashcat.net/hashcat/

https://passwordsgenerator.net/

https://stackoverflow.com/questions/45990454/generating-all-possible-combinations-of-characters-in-a-string

https://terokarvinen.com/2021/install-debian-on-virtualbox/?fromSearch=debian

https://terokarvinen.com/2022/cracking-passwords-with-hashcat/

https://terokarvinen.com/2023/eettinen-hakkerointi-2023/

https://wiki.debian.org/NvidiaGraphicsDrivers

https://www.youtube.com/watch?v=Ndm5t4sy8o0
