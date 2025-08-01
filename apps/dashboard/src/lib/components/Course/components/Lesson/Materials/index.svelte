<script lang="ts">
  import Box from '$lib/components/Box/index.svelte';
  import VideoUploader from '$lib/components/Course/components/Lesson/Materials/Video/Index.svelte';
  import {
    deleteLessonVideo,
    handleUpdateLessonMaterials,
    isLessonDirty,
    lesson,
    lessonByTranslation,
    lessons,
    uploadCourseVideoStore
  } from '$lib/components/Course/components/Lesson/store/lessons';
  import { course } from '$lib/components/Course/store';
  import TextField from '$lib/components/Form/TextField.svelte';
  import HtmlRender from '$lib/components/HTMLRender/HTMLRender.svelte';
  import IconButton from '$lib/components/IconButton/index.svelte';
  import Modal from '$lib/components/Modal/index.svelte';
  import { VARIANTS } from '$lib/components/PrimaryButton/constants';
  import PrimaryButton from '$lib/components/PrimaryButton/index.svelte';
  import { snackbar } from '$lib/components/Snackbar/store';
  import TabContent from '$lib/components/TabContent/index.svelte';
  import Tabs from '$lib/components/Tabs/index.svelte';
  import TextEditor from '$lib/components/TextEditor/index.svelte';
  import MODES from '$lib/utils/constants/mode';
  import { formatYoutubeVideo } from '$lib/utils/functions/formatYoutubeVideo';
  import { supabase } from '$lib/utils/functions/supabase';
  import { isHtmlValueEmpty } from '$lib/utils/functions/toHtml';
  import { lessonFallbackNote, t } from '$lib/utils/functions/translations';
  import { currentOrg } from '$lib/utils/store/org';
  import type { LessonPage, LOCALE } from '$lib/utils/types';
  import { useCompletion } from 'ai/svelte';
  import { Popover } from 'carbon-components-svelte';
  import AlignBoxTopLeftIcon from 'carbon-icons-svelte/lib/AlignBoxTopLeft.svelte';
  import IbmWatsonKnowledgeStudioIcon from 'carbon-icons-svelte/lib/IbmWatsonKnowledgeStudio.svelte';
  import ListIcon from 'carbon-icons-svelte/lib/List.svelte';
  import MagicWandFilled from 'carbon-icons-svelte/lib/MagicWandFilled.svelte';
  import TrashCanIcon from 'carbon-icons-svelte/lib/TrashCan.svelte';
  import isEmpty from 'lodash/isEmpty';
  import { fade } from 'svelte/transition';
  import Comments from './components/Comments.svelte';
  import ComponentNote from './components/ComponentNote.svelte';
  import ComponentSlide from './components/ComponentSlide.svelte';
  import ComponentVideo from './components/ComponentVideo.svelte';
  import * as CONSTANTS from './constants';
  import { orderedTabs } from './constants';
  import Loader from './Loader.svelte';

  export let mode = MODES.view;
  export let prevMode = '';
  export let lessonId = '';
  export let isSaving = false;
  export let isStudent = false;
  export let toggleMode = () => {};

  let localeExists: Record<string, boolean> = {};
  let lessonTitle = '';
  let prevContent = '';
  let initAutoSave = false;
  let timeoutId: NodeJS.Timeout;
  let tabs = CONSTANTS.tabs;
  let currentTab = tabs[0].value;
  let errors: Record<string, string> = {};
  let editorWindowRef: Window;
  let aiButtonRef: HTMLDivElement;
  let openPopover = false;
  let player: HTMLVideoElement;
  let componentsToRender = getComponentOrder(tabs);
  let aiButtonClass =
    'flex items-center px-5 py-2 border border-gray-300 hover:bg-gray-200 dark:hover:bg-gray-700 rounded-md w-full mb-2';

  const onChange = (tab) => {
    return () => {
      currentTab = tab;
    };
  };

  const getValue = (label: string) => {
    const tabValue = tabs.find((tab) => tab.label === label)?.value;
    return tabValue;
  };

  async function saveOrUpdateTranslation(locale, lessonId) {
    const content = $lessonByTranslation[lessonId][locale];

    if (typeof localeExists[locale] === 'undefined') {
      const { data } = await supabase
        .from('lesson_language')
        .select(`id`)
        .eq('lesson_id', lessonId)
        .eq('locale', locale)
        .maybeSingle();

      localeExists[locale] = !!(data && data?.id);
    }

    if (localeExists[locale]) {
      const { error: updateError } = await supabase
        .from('lesson_language')
        .update({ content })
        .eq('lesson_id', lessonId)
        .eq('locale', locale);

      if (updateError) {
        console.error('Error updating translation:', updateError.message);
        snackbar.error('snackbar.materials.update_translations');
      }
    } else {
      const { error: insertError } = await supabase.from('lesson_language').insert({
        locale,
        lesson_id: lessonId,
        content
      });

      if (insertError) {
        console.error('Error inserting translation:', insertError.message);
        snackbar.error('snackbar.materials.creating_new');
        return;
      }

      localeExists[locale] = true;
    }
  }

  async function saveLesson(materials?: LessonPage['materials']) {
    const _lesson = !!materials
      ? {
          ...$lesson,
          materials
        }
      : $lesson;

    console.log('updating lesson');
    const [lessonRes] = await Promise.all([
      handleUpdateLessonMaterials(_lesson, lessonId),
      saveOrUpdateTranslation($lesson.locale, lessonId)
    ]);

    return lessonRes;
  }

  function isMaterialsEmpty(
    materials: LessonPage['materials'],
    translation: Record<LOCALE, string>
  ) {
    const { slide_url, videos, note } = materials;

    return (
      isHtmlValueEmpty(note) &&
      !slide_url &&
      isEmpty(videos) &&
      Object.values(translation || {}).every((t) => isHtmlValueEmpty(t))
    );
  }

  function handleSave(prevMode: string) {
    if (prevMode === MODES.edit) {
      saveLesson();
    }
  }

  function addBadgeValueToTab(materials: LessonPage['materials']) {
    const { slide_url, videos, note } = materials;

    tabs = tabs.map((tab) => {
      let badgeValue = 0;

      if (tab.value === 1 && !isHtmlValueEmpty(note)) {
        badgeValue = 1;
      } else if (tab.value === 2 && !!slide_url) {
        badgeValue = 1;
      } else if (tab.value === 3 && !isEmpty(videos)) {
        badgeValue = 1;
      }
      tab.badgeValue = badgeValue;
      return tab;
    });
  }

  const openAddVideoModal = () => {
    $uploadCourseVideoStore.isModalOpen = true;
  };

  const { input, handleSubmit, completion, isLoading } = useCompletion({
    api: '/api/completion'
  });

  function updateNoteByCompletion(completion: string) {
    if (!completion) return;

    if ($lessonByTranslation[lessonId]) {
      $lessonByTranslation[lessonId][$lesson.locale] = `${prevContent}${completion}`;
    }

    autoSave($lesson.materials, $lessonByTranslation[lessonId], false, lessonId);

    if (editorWindowRef) {
      const tmceBody = editorWindowRef?.document?.querySelector('body');
      if (typeof tmceBody?.scrollHeight === 'number') {
        editorWindowRef?.scrollTo(0, tmceBody.scrollHeight);
      }
    }
  }

  function callAI(type = '') {
    prevContent = $lessonByTranslation[lessonId]?.[$lesson.locale] || '';

    const _lesson = $lessons.find((les) => les.id === $lesson.id);
    $input = JSON.stringify({
      type,
      lessonTitle: _lesson?.title || '',
      courseTitle: $course.title,
      locale: $lesson.locale
    });

    setTimeout(() => {
      handleSubmit({ preventDefault: () => {} });
    }, 500);
  }

  function initPlyr(_player: any, _video: LessonPage['materials']['videos']) {
    if (!_player) return;

    const players = Array.from(document.querySelectorAll('.plyr-video-trigger')).map((p) => {
      // @ts-ignore
      return new Plyr(p);
    });

    // @ts-ignore
    window.players = players;
  }

  function autoSave(
    updatedMaterials: LessonPage['materials'],
    translation: Record<LOCALE, string>,
    _isLoading?: boolean,
    lessonId?: string
  ) {
    if (mode === MODES.view) return;

    if (timeoutId) clearTimeout(timeoutId);

    if (!initAutoSave) {
      initAutoSave = true;
      return;
    }

    isSaving = true;
    timeoutId = setTimeout(async () => {
      const { error } = await saveLesson(updatedMaterials);

      if (error) {
        console.error('error saving lesson', error);
        snackbar.error('snackbar.materials.apology');
      }
      isSaving = false;
    }, 1000);
  }

  async function onLessonIdChange(_lid: string) {
    initAutoSave = false;
    isSaving = false;

    tabs = orderedTabs(tabs, $course.metadata?.lessonTabsOrder);
    currentTab = tabs[0].value;
    componentsToRender = getComponentOrder(tabs);
  }

  const onClose = () => {
    if ($uploadCourseVideoStore.isUploading) return;

    $uploadCourseVideoStore.isModalOpen = false;
    autoSave($lesson.materials, $lessonByTranslation[lessonId], $isLoading, lessonId);
  };

  function getComponentOrder(tabs = CONSTANTS.tabs) {
    const componentMap = {
      '1': ComponentNote,
      '2': ComponentSlide,
      '3': ComponentVideo
    };

    const componentNames = tabs
      .map((tab) => {
        const component = componentMap[tab.value];
        return component || null;
      })
      .filter(Boolean);

    return componentNames;
  }

  $: autoSave($lesson.materials, $lessonByTranslation[lessonId], $isLoading, lessonId);

  $: onLessonIdChange(lessonId);

  $: handleSave(prevMode);

  $: addBadgeValueToTab($lesson.materials);

  $: updateNoteByCompletion($completion);

  $: initPlyr(player, $lesson.materials.videos);

  $: lessonTitle = $lesson.title;

  $: editorValue = lessonFallbackNote(
    $lesson.materials.note,
    $lessonByTranslation[lessonId],
    $lesson.locale
  );
</script>

<Modal
  {onClose}
  bind:open={$uploadCourseVideoStore.isModalOpen}
  width="w-4/5 w-[90%] h-[80%] md:h-[566px]"
  modalHeading={$t('course.navItem.lessons.materials.tabs.video.add_video.title')}
>
  <VideoUploader {lessonId} />
</Modal>

<HtmlRender className="m-auto text-center">
  <svelte:fragment slot="content">
    <h1 class="mt-0 text-2xl capitalize md:text-4xl">
      {lessonTitle}
    </h1>
  </svelte:fragment>
</HtmlRender>

{#if $lesson.isFetching}
  <Loader />
{:else if mode === MODES.edit}
  <Tabs {tabs} {currentTab} {onChange}>
    <slot:fragment slot="content">
      <TabContent
        value={getValue('course.navItem.lessons.materials.tabs.note.title')}
        index={currentTab}
      >
        <div class="flex justify-end gap-1">
          <div bind:this={aiButtonRef} class="flex flex-row-reverse">
            <PrimaryButton
              className="flex items-center relative"
              onClick={() => {
                openPopover = !openPopover;
              }}
              isLoading={$isLoading}
              isDisabled={$isLoading}
              variant={VARIANTS.OUTLINED}
              disableScale
            >
              <MagicWandFilled size={20} class="carbon-icon mr-3" />
              AI
              <Popover
                caret
                align="left"
                bind:open={openPopover}
                on:click:outside={({ detail }) => {
                  openPopover = aiButtonRef?.contains(detail.target);
                }}
              >
                <div class="p-2">
                  <button class={aiButtonClass} on:click={() => callAI('outline')}>
                    <ListIcon class="carbon-icon mr-2" />
                    {$t('course.navItem.lessons.materials.tabs.note.ai.outline')}
                  </button>
                  <button class={aiButtonClass} on:click={() => callAI('note')}>
                    <AlignBoxTopLeftIcon class="carbon-icon mr-2" />
                    {$t('course.navItem.lessons.materials.tabs.note.ai.note')}
                  </button>
                  <button class={aiButtonClass} on:click={() => callAI('activities')}>
                    <IbmWatsonKnowledgeStudioIcon class="carbon-icon mr-2" />
                    {$t('course.navItem.lessons.materials.tabs.note.ai.activities')}
                  </button>
                </div>
              </Popover>
            </PrimaryButton>
          </div>
        </div>

        <div class="mt-5 h-[60vh]">
          <TextEditor
            id={lessonId}
            bind:editorWindowRef
            value={editorValue}
            onChange={(html) => {
              if (mode === MODES.view) return;
              $lessonByTranslation[lessonId][$lesson.locale] = html;
              try {
                // Backup locale of lesson content
                localStorage.setItem(`lesson-${lessonId}-${$lesson.locale}`, html);
              } catch (error) {}
              $isLessonDirty = true;
            }}
            placeholder={$t('course.navItem.lessons.materials.tabs.note.placeholder')}
          />
        </div>
      </TabContent>

      <TabContent
        value={getValue('course.navItem.lessons.materials.tabs.slide.title')}
        index={currentTab}
      >
        {#if mode === MODES.edit}
          <TextField
            label={$t('course.navItem.lessons.materials.tabs.slide.slide_link')}
            bind:value={$lesson.materials.slide_url}
            onInputChange={() => ($isLessonDirty = true)}
            helperMessage={$t('course.navItem.lessons.materials.tabs.slide.helper_message')}
          />
        {/if}
      </TabContent>
      <TabContent
        value={getValue('course.navItem.lessons.materials.tabs.video.title')}
        index={currentTab}
      >
        <PrimaryButton
          label={$t('course.navItem.lessons.materials.tabs.video.button')}
          onClick={openAddVideoModal}
          className="mb-2"
        />
        {#if $lesson.materials.videos.length}
          <div class="flex h-full w-full flex-col items-start">
            {#each $lesson.materials.videos as video, index}
              {#if mode === MODES.edit}
                <div class="ml-auto">
                  <IconButton
                    value="delete-video"
                    contained={true}
                    onClick={() => deleteLessonVideo(index)}
                  >
                    <TrashCanIcon size={20} class="carbon-icon dark:text-white" />
                  </IconButton>
                </div>
              {/if}
              <div class="flex h-full w-full flex-col gap-2 overflow-hidden">
                {#key video.link}
                  <div class="mb-5">
                    {#if video.type === 'youtube'}
                      <iframe
                        width="100%"
                        height="569"
                        class="iframe"
                        src={formatYoutubeVideo(video.link, errors)}
                        title="YouTube video player"
                        frameborder="0"
                        allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture"
                        allowfullscreen
                      />
                    {:else if video.type === 'generic'}
                      <iframe
                        width="100%"
                        height="569"
                        class="iframe"
                        src={video.link}
                        title="Embeded Video player"
                        frameborder="0"
                        allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture"
                        allowfullscreen
                      />
                    {:else if video.metadata?.svid}
                      <div style="position:relative;padding-bottom:51.416579%">
                        <iframe
                          src="https://muse.ai/embed/{video.metadata
                            ?.svid}?logo=https://app.classroomio.com/logo-512.png&subtitles=auto&cover_play_position=center"
                          style="width:100%;height:100%;position:absolute;left:0;top:0"
                          frameborder="0"
                          allowfullscreen
                          title="Muse AI Video Embed"
                        />
                      </div>
                    {:else}
                      <video bind:this={player} class="plyr-video-trigger" playsinline controls>
                        <source src={video.link} type="video/mp4" />
                        <track kind="captions" />
                      </video>
                    {/if}
                  </div>
                {/key}
              </div>
            {/each}
          </div>
        {/if}
      </TabContent>
    </slot:fragment>
  </Tabs>
{:else if !isMaterialsEmpty($lesson.materials, $lessonByTranslation[lessonId])}
  {#key lessonId}
    <div class="mb-20 w-full" in:fade={{ delay: 500 }} out:fade>
      {#each componentsToRender as Component}
        <svelte:component this={Component} {lessonId} />
      {/each}

      {#if $currentOrg.customization.apps.comments}
        <hr class="my-5" />
        <Comments {lessonId} />
      {/if}
    </div>
  {/key}
{:else}
  <Box className="text-center">
    <img src="/no-video.svg" alt="Video not found" />
    <h3 class="py-2 text-xl font-normal dark:text-white">
      {$t('course.navItem.lessons.materials.body_heading')}
    </h3>

    {#if !isStudent}
      <p class="py-2 text-center text-sm font-normal">
        {$t('course.navItem.lessons.materials.body_content')}
        <strong>{$t('course.navItem.lessons.materials.get_started')}</strong>
        {$t('course.navItem.lessons.materials.button')}.
      </p>
      <PrimaryButton
        label={$t('course.navItem.lessons.materials.get_started')}
        className="rounded-md"
        onClick={toggleMode}
      />
    {/if}
  </Box>
{/if}
