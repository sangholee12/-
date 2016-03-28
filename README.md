#Project 1 
#### 1. How to build Kernel
1.	Download prerequisites for Tizen_2.4 


2.	Download the our kernel source of Tizen kernel. 

	$ git clone https://github.com/swsnu/os-team2.git		

3. Change the branch to proj1

	$ cd os-team2
	$ git checkout proj1

4. Build kernel using build.sh file

	$ ./build.sh tizen_tm1 USR

#### 2. High-level Design & Implementation
1. SYSCALL ptree 구현

 1.1 Error Check
 - Syscall이 불리면 가장 먼저 Parameters에 대한 기본적인 에러체크 실행
 - buf, nr, buf-size 가 올바르면 0, 아니면 nonzero

	if (errcode = error_handling(buf,nr,&buf_size)){
		return errcode;
	}

 1.2 lock
 - task-list 를 돌면서 process의 정보를 가지고 오기전에 syscall이 불린 시점의 process만 고려하기 위해 고정 
 - 이 때 프로세스를 돌면서 visited array의 size를 정하게 된다. 

 1.3 Define visited array size
 - C언어에서는 기본적으로 array의 크기에 변수가 들어갈 수 없기 때문에, 이를 macro를 활용하여 preprocess 단계에서 define 해준다. 
 - 1.2에서 얻은 size는 process 들이 visited이 된지 확인하는 flag의 array인 visited array의 size가 된다. 


 	\#define DEFINE_VISITED(name, size)
 		char name[size]
	
	---
	
	DEFINE_VISITED(visited, pid_max+1);
	
 - 이제, dfs function 을 부르면서 탐색을 시작한다. 

 1.4 Call DFS function 
 - 이 함수에서는 프로세스 트리의 정보를 DFS pre-ordering 순서로 kernel memory 버퍼에 옮긴다.  
 - tasklist를 단순탐색하는 과정은

 	for_each_process(task) 

 라는 매크로를 활용한다. 
 - 여기서는 visited가 되지 않은 process에 대해서 aDFS를 부르게 된다. 즉 실제로는 제일 위에 있는 swapper에 바로 밑에 있는 child node에 대해 aDFS를 call한다. 

 	aDFS(task, entry_list, buf_size, visited, num_entry_cpy,num_entry_act);
 
 1.5 Call aDFS function
 - DFS의 auxiliary function
 - aDFS 는 recursive하게 불리는 function 으로 현재 탐색 중인 process 의 정보를 버퍼에 넘겨준다.
 - 버퍼에는 DFS pre-order로 프로세스 정보를 넘겨줘야 하므로, 불려진 프로세스를 쓰고 나면 first_child에 다시 aDFS가 불리게 된다. 
 - 이후에는 다음 child가 불리고, 최종적으로 부모의 자식 task list head와 자식의 sibling이 동일하면 다 돌았다는 뜻이므로 종료하게 된다.

<p align="center">
  <img src="http://forum.falinux.com/zbxe/files/attach/images/583/327/553/list_head.PNG" />
</p>

 - 만일 시스템콜이 불려질 때의 버퍼 사이즈보다 많은 프로세스가 존재한다면, 프로세스 정보를 카피하지는 않고, 개수만 늘려 나중에 돌려준다. 

 1.6 Copy to user
 - dfs를 통해 kernel 메모리에 DFS_preoreder로 쓰여진 프로세스 정보를 유저가 넘겨준 user memory에 써주고, 실제 프로세스의 개수를 넘겨주고 끝나게 된다. 

2. Test
 - 만들어진 시스템콜을 부르고, indent를 하며 프린트한다. 
 
#### 3. Investigation of the process tree
3.1 test 프로그램을 단순히 여러번 불렀을 때.
 test프로세스: pid만 변한다. 나머지는 변하지 않는다 
 이는 test가 불리고 꺼지면서 test를 부르는 부모 프로세스에서 test를 부를 때 마다, 새로운 프로세스를 생성하기 때문에 변하는 것으로 보인다. 
  
	 < 		sh,1949,1,959,2066,0,5100
	 < 			test,2066,0,1949,0,0,5100
	 ---
	 > 		sh,1949,1,959,2082,0,5100
	 > 			test,2082,0,1949,0,0,5100

3.2.1 카메라 실행 후
- 바뀌는 것
	test: pid만 변한다. 
	launchpad-loader (pid:990): camera (pid:990)으로 변한다. 
	ext4-dio-unwrit: next_sibling이 0에서 1239로 바뀐다. 

- 추가되는 것
	kworker/0:3 
	dcam_flash_thread
	img_zoom_thread
	ipp_cmd_1 
- 추가되는 프로세스는 모두 kthread (pid:2)를 부모로 가진다. 
  ext4-dio는 마지막 자식으로 존재하다가 카메라를 실행하면서 위의 추가되는 프로세스들이 생성되면서 위의 프로세스들을 sibling으로 가지게된다. 
- launchpad-loader의 변화: 3.3 에서 다룬다. 

3.2.2 카메라 실행 후 홈 버튼을 누르고 난 뒤. 
- 바뀌는 것
	test: 마찬가지로 pid가 바뀐다. 
 	kworker/0:3 (pid:1239) : next_sibling 이 0으로 바뀐다.

- 없어지는 것
	dcam_flash
	imag_zoom_thread
	ipp_cmd

- camera(pid:990)의 프로세스는 그대로 남아있지만, 위의 세 프로세스는 사라지게 된다.
- 이를 통해 홈 버튼을 누르는 것이 카메라 프로세스는 여전히 백그라운드에서 돌아가고, 카메라와 관련된 추가적인 프로세스는 종료되는 것을 확인 할 수 있다.


3.3 launchpad, launchpad-loader
- launchpad-loader는 모두 launchpad를 부모로 가진다.
- 카메라 혹은 다른 앱을 실행하면, launchpad-loader의 프로세스는 해당 프로세스로 exec되어 이름이 바뀌고 pid는 그대로 유지된다. 

- lauchpad-loader는 일종의 dummy 프로세스로 유저 어플리케이션을 실행시키기 위한 프로세스로 존재한다. 이를 통해 어플리케이션을 실행시키면, 프로세스를 다시 만들고 할당하는 것이 아니라, 이미 존재하는 loader의 프로세스를 exec만하게 된다.
- 새로 만드는 것이 아니므로, 처음부터 새로 프로세스를 만드는 과정보다 속도가 빠르다.
- launchpad process 는 이러한 loader를 만드는 프로세스인데, 유저가 실행하는 프로세스는 loader에서 exec되므로, 유저 프로세스를 생성/종료 등의 관리를 할 수 있어, 메모리 관리 측면에서도 장점이 있다. 


#### 4. Lesson Learned 
- 스마트폰에서 필요한 프로세스
	test 프로그램을 여러 환경에서 실행한 결과, test 프로세스 말고도 여러 프로세스가 바뀌는 것을 관찰 할 수 있었다. 
	가령, 화면이 꺼져있는 상태와 잠금화면인 상태, 그리고 홈화면인 상태일 때의 프로세스가 모두 다르고, 그 때마다 필요한 프로세스가 생성/종료되는 것을 확인 할 수 있었다. 
	또한 화면을 키고 몇 번 화면을 움직이기 전과 후를 비교할 때도, 아예 똑같지는 않았는데, 이를 통해 유저 입장에서는 단순한 행동이 얼마나 많은 프로세스를 부르고 종료시키는 지를 알 수 있었다. 

- launchpad의 존재
	여태까지는 어플리케이션을 실행시키면 단순히 프로세스가 생성되는 줄 알았으나, 막상 프로세스 트리를 관찰하여 보니, launchpad를 통해 프로세스가 관리되고 생성되는 것을 알 수 있었다. 

- 어플리케이션 실행
	카메라 어플리케이션을 실행한 결과, 카메라 프로세스 자체는 launchpad의 자식으로 존재하지만, 이와 같이 실행되는 flash, zoom-in 과 같은 프로세스는 systemd 밑에 있는 것이 아니라, kthreadd 밑에 존재하는 것을 확인 할 수 있었고다.
	가령, 홈화면을 눌렀을 때, 카메라 프로세스는 존재하고 나머지는 따로 종료되는 것 처럼, 카메라 프로세스 자체와 다르게 관리되는 것을 알 수 있었다. 
