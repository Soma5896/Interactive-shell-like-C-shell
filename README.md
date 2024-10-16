# Interactive-shell-like-C-shell
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <sys/wait.h>
#include <signal.h>
#include <fcntl.h>
#include <sys/types.h>
#include <pwd.h>

#define MAX_INPUT_SIZE 1024
#define MAX_ARGS 64

int foreground = 1;
char prompt[MAX_INPUT_SIZE];

void sigint_handler(int signum) {
    // Handle SIGINT (Ctrl+C)
    printf("\nReceived SIGINT. Exiting...\n");
    
}

void sigtstp_handler(int signum) {
    // Handle SIGTSTP (Ctrl+Z)
    if (foreground) {
        printf("\nCtrl+Z pressed. Running in background...\n");
        foreground = 0; // Switching to background mode
    } else {
        printf("\nRunning in foreground...\n");
        foreground = 1; // Switching to foreground mode
    }
}

void setup_prompt() {
    char hostname[MAX_INPUT_SIZE];
    gethostname(hostname, MAX_INPUT_SIZE);
    struct passwd *pw = getpwuid(getuid());
    if (pw != NULL)
        sprintf(prompt, "%s@%s%% ", pw->pw_name, hostname);
    else
        sprintf(prompt, "ish%% ");
}

int main() {
    char input[MAX_INPUT_SIZE];
    char *args[MAX_ARGS];
    int status;

    // Registering signal handlers
    signal(SIGINT, sigint_handler);
    signal(SIGTSTP, sigtstp_handler);
     
    setup_prompt();
    
    // Read from .ishrc file
    FILE *ishrc = fopen(".ishrc", "r");
    if (ishrc) {
        char line[MAX_INPUT_SIZE];
        while (fgets(line, sizeof(line), ishrc)) {
            // Execute commands from .ishrc file
            system(line);
        }
        fclose(ishrc);
    }


    while (1) {
        printf("%s", prompt);
        fflush(stdout);

        // Reading input
        fgets(input, MAX_INPUT_SIZE, stdin);
        input[strcspn(input, "\n")] = '\0'; // Removing newline character

        // Tokenizing input
        char *token = strtok(input, " ");
        int arg_count = 0;
        while (token != NULL && arg_count < MAX_ARGS - 1) {
            args[arg_count++] = token;
            token = strtok(NULL, " ");
        }
        args[arg_count] = NULL;

        if (arg_count == 0)
            continue;

        // Checking for built-in commands
        if (strcmp(args[0], "exit") == 0)
            exit(0);
        else if (strcmp(args[0], "cd") == 0) {
            if (arg_count < 2)
                printf("Usage: cd <directory>\n");
            else if (chdir(args[1]) != 0)
                perror("cd");
            continue;
        } else if (strcmp(args[0], "jobs") == 0) {
            // Displaying background jobs
            printf("Foreground: PID=%d\n", getpid());
            printf("Background: No jobs\n");
            continue;
        } else if (strcmp(args[0], "kill") == 0) {
            // Sending SIGTERM signal to the specified PID
            if (arg_count < 2)
                printf("Usage: kill <PID>\n");
            else {
                int pid = atoi(args[1]);
                if (kill(pid, SIGTERM) != 0)
                    perror("kill");
            }
            continue;
        }else if (strcmp(args[0], "setenv") == 0) {
            if (arg_count < 2) {
                printf("Usage: setenv <name> <value>\n");
            } else if (setenv(args[1], args[2], 1) != 0) {
                perror("setenv");
            }
            continue;
        } else if (strcmp(args[0], "unsetenv") == 0) {
            if (arg_count < 2) {
                printf("Usage: unsetenv <name>\n");
            } else if (unsetenv(args[1]) != 0) {
                perror("unsetenv");
            }
            continue;
        }

        // Checking for background execution
        if (strcmp(args[arg_count - 1], "&") == 0) {
            foreground = 0;
            args[arg_count - 1] = NULL;
        } else {
            foreground = 1;
        }

        // Fork process
        pid_t pid = fork();
        if (pid < 0) {
            perror("fork");
            exit(1);
        } else if (pid == 0) {
            // Child process

            // Redirection and Pipe handling
            int in_fd, out_fd;
            for (int i = 0; args[i] != NULL; i++) {
                if (strcmp(args[i], "<") == 0) {
                    args[i] = NULL;
                    in_fd = open(args[i + 1], O_RDONLY);
                    if (in_fd == -1) {
                        perror("open");
                        exit(1);
                    }
                    if (dup2(in_fd, STDIN_FILENO) == -1) {
                        perror("dup2");
                        exit(1);
                    }
                    close(in_fd);
                } else if (strcmp(args[i], ">") == 0) {
                    args[i] = NULL;
                    out_fd = open(args[i + 1], O_WRONLY | O_CREAT | O_TRUNC, 0644);
                    if (out_fd == -1) {
                        perror("open");
                        exit(1);
                    }
                    if (dup2(out_fd, STDOUT_FILENO) == -1) {
                        perror("dup2");
                        exit(1);
                    }
                    close(out_fd);
                    } else if (strcmp(args[i], ">>") == 0) {
                    args[i] = NULL;
                    out_fd = open(args[i + 1], O_WRONLY | O_CREAT | O_APPEND, 0644);
                    if (out_fd == -1) {
                        perror("open");
                        exit(1);
                    }
                    if (dup2(out_fd, STDOUT_FILENO) == -1) {
                        perror("dup2");
                        exit(1);
                    }
                    close(out_fd);
                } else if (strcmp(args[i], "&>") == 0) {
                    args[i] = NULL;
                    out_fd = open(args[i + 1], O_WRONLY | O_CREAT | O_TRUNC, 0644);
                    if (out_fd == -1) {
                        perror("open");
                        exit(1);
                    }
                    if (dup2(out_fd, STDOUT_FILENO) == -1) {
                        perror("dup2");
                        exit(1);
                    }
                    if (dup2(out_fd, STDERR_FILENO) == -1) {
                        perror("dup2");
                        exit(1);
                    }
                    close(out_fd);
                } else if (strcmp(args[i], ">>&") == 0) {
                    args[i] = NULL;
                    out_fd = open(args[i + 1], O_WRONLY | O_CREAT | O_APPEND, 0644);
                    if (out_fd == -1) {
                        perror("open");
                        exit(1);
                    }
                    if (dup2(out_fd, STDOUT_FILENO) == -1) {
                        perror("dup2");
                        exit(1);
                    }
                    if (dup2(out_fd, STDERR_FILENO) == -1) {
                        perror("dup2");
                        exit(1);
                    }
                    close(out_fd);
                } else if (strcmp(args[i], "|") == 0) {
                    args[i] = NULL;

                    // Create pipes
                    int pipefd[2];
                    if (pipe(pipefd) == -1) {
                        perror("pipe");
                        exit(1);
                    }

                    // Fork process for the next command
                    pid_t next_pid = fork();
                    if (next_pid < 0) {
                        perror("fork");
                        exit(1);
                    } else if (next_pid == 0) {
                        // Child process for the next command

                        // Redirect stdin to read from the pipe
                        if (dup2(pipefd[0], STDIN_FILENO) == -1) {
                            perror("dup2");
                            exit(1);
                        }
                        close(pipefd[0]);
                        close(pipefd[1]);

                        // Execute the next command
                        execvp(args[i + 1], &args[i + 1]);
                        perror("execvp");
                        exit(1);
                    } else {
                        // Parent process

                        // Redirect stdout to write to the pipe
                        if (dup2(pipefd[1], STDOUT_FILENO) == -1) {
                            perror("dup2");
                            exit(1);
                        }
                        close(pipefd[0]);
                        close(pipefd[1]);

                        // Execute the first command
                        execvp(args[0], args);
                        perror("execvp");
                        exit(1);
                    }
                }
            }

            // Execute the command without pipes
            execvp(args[0], args);
            perror("execvp");
            exit(1);
        } else {
            // Parent process
            if (foreground) {
                waitpid(pid, &status, 0);
            } else {
                // Print background job information
                printf("[%d] %d\n", pid, getpid());
            }
        }
    }

    return 0;
}
