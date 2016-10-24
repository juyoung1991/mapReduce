//#define _GNU_SOURCE

#include <unistd.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include "common.h"


int main(int argc, char ** argv){
	if(argc < 6){
		printf("not enough arguments\n");
		exit(1);
	}
	FILE * outputFile = fopen(argv[2], "r");
	int count = atoi(argv[5]); // number of mappers
	int fd[count][2]; //mappers
	pid_t childSplit[count];
	pid_t childMap[count];
	pid_t childReduce;
	int fd2[2]; //reducer
	int i;
//===============================================================SPLITTER===================================================================
	for(i = 0; i < count; i++){
		pipe(fd[i]);
		char * splitArgv[5];
		splitArgv[0] = strdup("./splitter");
		splitArgv[1] = strdup(argv[1]);
		splitArgv[2] = strdup(argv[5]);
		splitArgv[3] = malloc(sizeof(int)+1);
		sprintf(splitArgv[3], "%d", i);
		splitArgv[4] = NULL;

		childSplit[i] = fork();
		if(childSplit[i] > 0){
			close(fd[i][1]);
		}else{
			dup2(fd[i][1], 1);
			close(fd[i][0]);
			close(fd[i][1]);
			execvp(splitArgv[0], splitArgv);
			exit(1);
		}
		int j;
		for(j = 0; j < 4; j++){
			free(splitArgv[j]);
		}
	}

//===============================================================MAPPER===================================================================
	pipe(fd2);
	for(i = 0; i < count; i++){
		childMap[i] = fork();
		if(childMap[i] > 0){
			close(fd[i][0]); //now close the reader
		}else{
			close(fd2[0]);
			dup2(fd[i][0], 0);
			close(fd[i][0]);
			dup2(fd2[1], 1);
			close(fd2[1]);

			execl(argv[3], argv[3], NULL);
			exit(1);
		}
	}

//===============================================================REDUCER===================================================================

	

	childReduce = fork();
	if(childReduce > 0){
		close(fd2[1]);
		close(fd2[0]);
	}else{
		dup2(fd2[0], 0);
		close(fd2[0]);
		close(fd2[1]);
		int ret = open(argv[2], O_CREAT | O_RDWR | O_TRUNC , S_IRUSR | S_IWUSR);
		dup2(ret, 1);
		execl(argv[4], argv[4], NULL);
		exit(1);
	}


//===============================================================WAIT FOR EVERYTHING===================================================================
	int status;
	for(i = 0; i < count; i++){
		waitpid(childSplit[i], &status, 0);
		
	}
	for(i = 0; i < count; i++){
		waitpid(childMap[i], &status, 0);
		if(status){
			printf("%s %d exited with status %d\n", argv[3], i, status);
		}
	}
	waitpid(childReduce, &status, 0);
	
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
