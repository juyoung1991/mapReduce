#include <stdio.h>
#include <stdlib.h>
#include <alloca.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <sys/wait.h>
#include <assert.h>
#include "common.h"


int main(int argc, char ** argv) {
	int num = argc-1;
	int i;
	int output_files[num];
	for(i = 0; i < num; i++){
		output_files[i] = open(argv[i+1], O_WRONLY);
	}
	size_t buffSize;
	char * buff = NULL;
	ssize_t line;
	//printf("here\n");
	while((line = getline(&buff, &buffSize, stdin)) != -1){
		//FROM CORE FILE
		char * split = strstr(buff, ": ");
        if (!split) {
            fprintf(stderr, "reducer input is malformed: %s\n", buff);
            continue;
        }
        *split = '\0';
        char * key = buff;
        char * value = split + 2; // account for the extra space
        value[strlen(value)-1] = '\0';
		int outf = output_files[hashKey(key) % num];
		dprintf(outf, "%s: %s\n", key, value);
	}
	free(buff);
    return 0;
}
