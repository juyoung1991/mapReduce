#include <unistd.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <sys/wait.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>
#include "common.h"


int main(int argc, char ** argv) {
	if(argc < 7){
		printf("not enough arguments\n");
		exit(1);
	}
	FILE * outputFile = fopen(argv[2], "r");
	int count = atoi(argv[5]); // number of mappers
	int redCnt = atoi(argv[6]); //number of reducers

	int fdMap[count][2]; //mapper pipes
	int fdShuff[2]; //shuffle pipe
	//int fdRed[redCnt][2]; //reducer pipes

	pid_t childSplit[count];
	pid_t childMap[count];
	pid_t childShuffle;
	pid_t childReduce[redCnt];
	//pid_t childLast;
	int i;
//===============================================================SPLITTER - MAPPER===================================================================
	for(i = 0; i < count; i++){
		pipe(fdMap[i]);
		char * splitArgv[5];
		splitArgv[0] = strdup("./splitter");
		splitArgv[1] = strdup(argv[1]);
		splitArgv[2] = strdup(argv[5]);
		splitArgv[3] = malloc(50);
		sprintf(splitArgv[3], "%d", i);
		splitArgv[4] = NULL;

		childSplit[i] = fork();
		if(childSplit[i] > 0){
			close(fdMap[i][1]);
		}else{
			dup2(fdMap[i][1], 1); //stdout
			close(fdMap[i][0]);
			close(fdMap[i][1]);
			execvp(splitArgv[0], splitArgv);
			exit(1);
		}
		int j;
		for(j = 0; j < 4; j++){
			free(splitArgv[j]);
		}
	}

//===============================================================MAPPER - SHUFFLER===================================================================
	pipe(fdShuff);
	for(i = 0; i < count; i++){
		childMap[i] = fork();
		if(childMap[i] > 0){
			close(fdMap[i][0]);
			//close(fdShuff[1]);
		}else{
			dup2(fdMap[i][0], 0); //stdin
			close(fdMap[i][0]);
			close(fdMap[i][1]);
			dup2(fdShuff[1], 1); //stdout
			close(fdShuff[1]);
			close(fdShuff[0]);
			//fprintf(stderr, "About to exec %s %d reading from fdMap[%d][0]=%d, writing to fdShuff[1]=%d\n", argv[3], i, i, fdMap[i][0], fdShuff[1]);
			execl(argv[3], argv[3], NULL);
			perror("mapper failed");
			exit(1);
		}
	}
	//close(fdMap[i][0]);
	close(fdShuff[1]);
//===============================================================SHUFFLER - FILES==================================================================	
	char * shuffArgv[redCnt+2];
	shuffArgv[0] = strdup("./shuffler");
	for(i = 0; i < redCnt; i++){
		char * file_name = malloc(50);
		sprintf(file_name, "./fifo_%d", i);
		mkfifo(file_name, S_IRWXU);
		shuffArgv[i+1] = strdup(file_name);
		free(file_name);
	//	free(itoa);
	}
	shuffArgv[redCnt+1] = NULL;
	/*
	printf("shuffArgv[0] = %s\n", shuffArgv[0]);
	printf("shuffArgv[1] = %s\n", shuffArgv[1]);
	printf("shuffArgv[2] = %s\n", shuffArgv[2]);
	printf("shuffArgv[3] = %s\n", shuffArgv[3]);
	printf("shuffArgv[4] = %s\n", shuffArgv[4]);
	printf("shuffArgv[5] = %s\n", shuffArgv[5]);*/
	childShuffle = fork();
	if(childShuffle > 0 ){
		close(fdShuff[0]);
	}else{
		dup2(fdShuff[0], 0); //stdin which comes from the mapper-shuffler pipe
		close(fdShuff[0]);
		//close(fdShuff[1]);
		execvp(shuffArgv[0], shuffArgv);
		exit(1);
	}

//===============================================================FILES - REDUCER===================================================================

	
	for(i = 0; i < redCnt; i++){
		childReduce[i] = fork();
		if(childReduce[i] > 0){
			//what should i do in parent??
			
		}else{
			int ret = open(shuffArgv[i+1], O_RDONLY);
			//printf("ret : %d\n", ret);
			dup2(ret, 0); //stdin
			close(ret);
			
			int out = open(argv[2], O_APPEND | O_CREAT | O_WRONLY , S_IRUSR | S_IWUSR);
			//printf("shuffler %d is input opening %s as FD %d and output %s\n", i, shuffArgv[i+1], ret, argv[2]);
			dup2(out, 1); //stdout
			close(out);
			execl(argv[4], argv[4], NULL);
			perror("error in reducer\n");
			exit(1);
		}
	}
	
	for(i = 0; i < redCnt + 1; i++){
		free(shuffArgv[i]);
	}
	
//===============================================================WAIT FOR EVERYTHING===================================================================
	int status;
	for(i = 0; i < count; i++){
		waitpid(childSplit[i], &status, 0);
		
	}
	//puts("split wait");
	for(i = 0; i < count; i++){
		waitpid(childMap[i], &status, 0);
		if(status){
			printf("%s %d exited with status %d\n", argv[3], i, status);
		}
	}
	
	//puts("mapper wait");
	waitpid(childShuffle, &status, 0);
	//puts("shuffle wait");
    for(i = 0; i < redCnt; i++){
		waitpid(childReduce[i], &status, 0);
		if(status){
			printf("%s %d exited with status %d\n", argv[4], i, status);
		}
	}
	//puts("reducer wait");

//===============================================================REMOVE FIFO===========================================================================
    // Remove the fifo files.
	//printf("here\n");

	for(i = 0; i < redCnt; i++){
		char * file_name = malloc(50);
		sprintf(file_name, "./fifo_%d", i);
		remove(file_name);
		free(file_name);
	}

//===============================================================FINISH================================================================================
	size_t buffSize;
	char * buff = NULL;
	ssize_t line;
	int linecount = 0;
	if(outputFile != NULL){
		fseek(outputFile, 0, SEEK_SET);
		while( (line = getline(&buff, &buffSize, outputFile)) != -1){
			linecount++;
		}
		free(buff);
		printf("output pairs in %s: %d\n", argv[2], linecount);
		fclose(outputFile);
	}else{
		outputFile = fopen(argv[2], "r");
		fseek(outputFile, 0, SEEK_SET);
		while( (line = getline(&buff, &buffSize, outputFile)) != -1){
			linecount++;
		}
		free(buff);
		printf("output pairs in %s: %d\n", argv[2], linecount);
		fclose(outputFile);
	}
    return 0;
}
