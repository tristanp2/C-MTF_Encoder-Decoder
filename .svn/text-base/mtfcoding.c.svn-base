#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include "mtfcoding.h"

#define INITSIZE 120
#define MAXPILESIZE 65912

typedef struct{
    char* word;
    int space;
}Word;

//This should be in the header, but we are only allowed to
//modify this file
void* mem_check(void*,int);

const char OLDMAGIC[]={0xfa,0xce,0xfa,0xde};
const char MAGIC[]={0xfa,0xce,0xfa,0xdf,'\0'};
Word* pile; //This serves as the pile or dictionary of words
int pileHeight;
int pileSpace=INITSIZE;

//Initializes the pile
void init_pile()
{
    pileHeight=0;
    pileSpace=INITSIZE;
    pile=mem_check(NULL,INITSIZE*sizeof(Word));
}
void debug_pile_dump()
{
    int i;
    for(i=0;i<pileHeight;i++)
        printf("[%i]\t%s\n",i,pile[i].word);
}
//This function can be used to reallocate or allocate new memory
//To allocate new memory, NULL should be passed in for loc
//The size parameter should be given in bytes
void* mem_check(void* loc, int size)
{    
    void* check;
    check=realloc(loc,size);
    if(check==NULL)
    {
        puts("ERROR. Could not allocate memory.");
        exit(1);
    }
    return check;
}
//Moves the word at index to the top of the pile
//Memory is only reallocated if the copy destination 
//has less space than the source
void move_to_top(int index)
{   
    int len=strlen(pile[index].word)+1;
    char temp[len];
    int i;
    strcpy(temp,pile[index].word);
    for(i=index+1;i<pileHeight;i++)
    {
        if(pile[i-1].space<pile[i].space)
        {
            pile[i-1].word=(char*)mem_check(pile[i-1].word,pile[i].space);
            pile[i-1].space=pile[i].space;
        }
        strcpy(pile[i-1].word,pile[i].word);        
    }
    if(pile[i-1].space<len)
    {
        pile[i-1].word=(char*)mem_check(pile[i-1].word,len);
        pile[i-1].space=len;
    }
    strcpy(pile[i-1].word,temp);            
}
//Called when the current memory block of the pile is
//completely full
void grow_pile()
{
    pileSpace*=2;
    pile=(Word*)mem_check(pile,pileSpace*sizeof(Word));
}
//Allocates memory for the word and adds it to the pile
void add_to_pile(char* word)
{
    int len=strlen(word)+1;
    if(pileHeight==pileSpace)
        grow_pile();
    pile[pileHeight].word=mem_check(NULL, len);
    pile[pileHeight].space=len;
    strcpy(pile[pileHeight].word, word);
    pileHeight++;
}
//Outputs the code corresponding to index to
//If index is -1, then the code for a new
//word is output
void output_code(int index, FILE* output)
{
    if(index==-1)
    {
        if(pileHeight<=120)
            fprintf(output, "%c", (char)(pileHeight+128));
        else if(pileHeight<=375)
            fprintf(output, "%c%c", (char)0xf9, (char)(pileHeight-121));
        else
            fprintf(output, "%c%c%c", 0xfa, (pileHeight-376)/256, (pileHeight-376)%256);
    }
    else
    {
        if(pileHeight-index<=120)
            fprintf(output, "%c", pileHeight-index+128);
        else if(pileHeight-index<=375)
            fprintf(output, "%c%c", 0xf9, pileHeight-index-121);
        else
            fprintf(output, "%c%c%c", 0xfa, (pileHeight-index-376)/256, (pileHeight-index-376)%256);
    }
}
//Checks pile to see if word is repeated or new
//If new, the new code is outputted followed by the word
//If repeated, the code corresponding to its position
//in the pile is outputted
int encode_word(char *word, FILE* output)
{
    int i;
    for(i=0; i<pileHeight; i++)
    {
        if(!strcmp(word,pile[i].word))
            break;
    }
    if(i==pileHeight)
    {
        if(pileHeight==MAXPILESIZE)
        {
            puts("The pile is too full to add another word. Exiting...");
            exit(2);
        }
        add_to_pile(word);
        output_code(-1,output);
        fprintf(output,"%s",word);
    }
    else
        output_code(i,output);
    return i;
}
//Uses strtok to extract words from line and encode them
void encode_then_out(ssize_t length, char *line, FILE *output)
{
    char temp[length];
    char* t;
    int index;

    strcpy(temp,line);
    t=strrchr(temp,'\n');
    if(t)
        *t='\0'; //replacing newline with null
    
    t=strtok(temp," ");
    while(t)
    {
        index=encode_word(t, output);
        move_to_top(index);
        t=strtok(NULL," ");
    }
}
//The main function of the encoder
int encode(FILE *input, FILE *output)
{
    init_pile();
    char* line=NULL;
    size_t len=0;
    ssize_t read;
    
    fseek(input,0,SEEK_END);
    int filesize=ftell(input);
    rewind(input);

    fprintf(output,"%s",MAGIC); 
    while((read=getline(&line, &len, input))>0)
    {
        encode_then_out(read, line, output);
        fprintf(output,"\n");
        printf("\r%.2f%% complete",((float)ftell(input)/filesize)*100);
    }
    printf("\r%.2f%% complete\n",((float)ftell(input)/filesize)*100);
    return 0;
}

//This function is used to extrace words from an mtf file,
//as strtok would not work
Word get_word(FILE *f)
{
    int start=ftell(f);
    int ch;
    Word str;
    while(1)
    {
        ch=fgetc(f);
        if(ch>0x80 || ch=='\n')
            break;
    }
    int end=ftell(f);
    str.space=end-start;
    fseek(f,start,0);
    str.word=mem_check(NULL,str.space);
    fread(str.word,str.space,sizeof(char),f);
    str.word[str.space-1]='\0';
    fseek(f,end-1,0);
    //printf("String found: %s\n",str.word);
    return str;    
}
//The main function of the decoder
//The get_word function is used in the loop in such a way 
//that inchar will always be a code byte or newline
//Once the code is calculated, the process of modifying the pile
//and outputting the text is identical for all three code sizes
int decode(FILE *input, FILE *output)
{
    char headcheck[4];
    fread(headcheck,4,sizeof(char),input);
    //The following checks the first 4 bytes to ensure that it has been
    //encoded with this coding scheme
    //strncmp returns 0 when the strings are equal, so the statement
    //will only be true if neither magic number is equal to headcheck
    if(strncmp(headcheck,MAGIC,4) && strncmp(headcheck,OLDMAGIC,4))
        return -1;
    
    fseek(input,0,SEEK_END);
    int filesize=ftell(input);
    fseek(input,4,0);
    void* check;
    init_pile();
    int code,inchar,index;
    Word buff;
    while((inchar=fgetc(input))!=EOF)
    {
        if(inchar=='\n')      //This handles empty lines
        {
            fprintf(output,"\n");
        }
        else if(inchar>0x80)    //This if statement probably isn't necessary
        {
            if(inchar<=0xf8)        //1 byte code
            {
                code=inchar-0x80;
            }
            else if(inchar==0xf9)   //2 byte code
            {
                inchar=fgetc(input);
                code=inchar+121;
            }
            else if(inchar==0xfa)   //3 byte code
            {
                inchar=fgetc(input);
                code=inchar*256+(int)fgetc(input)+376;
            }

            if(code>pileHeight)
            {
                free(buff.word);
                buff=get_word(input);     
                add_to_pile(buff.word);
            }
            else
            {
                index=pileHeight-code;
                if(buff.space<pile[index].space)
                {
                    buff.word=mem_check(buff.word,pile[index].space);
                }
                strcpy(buff.word,pile[index].word);
                move_to_top(index);
            }
            fprintf(output,"%s",buff.word);
            //If the next char is a newline, it will be outputted
            //else the char must be handled at the start of
            //the next iteration
            if(fgetc(input)=='\n')
                fprintf(output,"\n");
            else
            {
                fprintf(output," ");
                fseek(input,ftell(input)-1,0);
            }
        }
        printf("\r%.2f%% complete",((float)ftell(input)/filesize)*100);
    }
    int i;
    printf("\r%.2f%% complete\n",((float)ftell(input)/filesize)*100);
    return 0;
}
