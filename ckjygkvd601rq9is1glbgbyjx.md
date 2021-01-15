## Bash: writing a simple pod checker

> I was working with Kubernetes and I just want to check my pods in the current workspace, in a funny way  😅, I will use bash for just one reason, it's the Linux native languages no need for other languages and libraries


Actually, I was inspired by a cool tool called **popeye**, you can find it [here](https://github.com/derailed/popeye). 

Make sure you have **Kubectl** installed and an existing cluster as well, here is my implementation 

* **Define your favorite colors**

```bash
blanc="\033[1;37m"
gray="\033[0;37m"
magento="\033[0;35m"
red="\033[1;31m"
green="\033[1;32m"
amarillo="\033[1;33m"
azul="\033[1;34m"
rescolor="\e[0m"
```

You can find more about colors in bash [here](https://misc.flogisoft.com/bash/tip_colors_and_formatting).

* **Get list pods into an Array**

```bash
listPods=$(kubectl get po | awk 'NR>1{print $1}')
#echo "$listPods"
readarray  arr <<<  $listPods
```
`NR>1` will skip the first line and `print $1` will print the first words (separated by a space) on all lines.

* **Looping over the array and check the status**

```bash
ok=0
notok=0
echo -e "\nSit Down and Wait  \U1F602 :\n"
for i in ${arr[@]}
do 
echo -ne "$i ... " 
status=$(kubectl get po $i | grep $i | awk '{print $3}')
	if [[ ! $status =~ ^Running$|^Completed$  ]]  ; then
		echo -e "\e[1;31mOh Shit !"$rescolor""
        notify-send "Pods Health" "$i was  FUCKED" -t 10000 
        let notok=notok+1
	else
		echo -e "\e[1;32mOK!"$rescolor""
        #notify-send "Pod $i Is Good :)"
      let ok=ok+1
	fi
done
```

The `ok` and `notok` are used to count the number of the running/not running pods , the `${arr[@]}` prints out the whole array, the `notify-send` will create a notification on your system is one of the pods are **f**ked up** .

* **Print out the summary**

```bash
echo -e "\nSTATS:\n"
echo "+---------------+---------------+"
printf  "|$green%-15s$rescolor|$red%-15s$rescolor|\n" "Healthy Pods" "Unhealthy Pods"
echo "+---------------+---------------+"
printf  "|%-15s|%-15s|\n" "$ok" "$notok"
echo "+---------------+---------------+"
echo -e "\n"
```

* **Run the script**

Download the scripts with `CURL` : 
```bash
curl https://raw.githubusercontent.com/hatembentayeb/podschecker/master/podschecker.sh --output podschecker.sh
chmod +x podschecker.sh
```

* **Demo** 

![PodHealth.gif](https://cdn.hashnode.com/res/hashnode/image/upload/v1610725608108/yrhW96uAh.gif)


***The scripts don't do much but it was a result of a boring 3 hours on this pandemic**  🥲
