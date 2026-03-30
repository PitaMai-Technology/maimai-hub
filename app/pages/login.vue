<script setup lang="ts">
import type { FormSubmitEvent } from '@nuxt/ui';
import { z } from 'zod';
import { authClient } from '~/composable/auth-client';
import { useTurnstile } from '~/composable/useTurnstile';

definePageMeta({
  layout: 'the-front',
});

const toast = useToast();
const loading = ref(false);
const otpSent = ref(false);
const oauthLoading = ref(false);
const loginMethod = ref<'email' | 'oauth'>('oauth');
const session = authClient.useSession();
const { config, turnstileToken, resetTurnstileToken } = useTurnstile('login-turnstile');

const emailState = reactive({
  email: '',
});

const otpState = reactive({
  otp: '',
});

const emailOtpFormSchema = z.object({
  email: z.email('有効なメールアドレスを入力してください'),
});

const emailOtpVerifySchema = z.object({
  otp: z.string().regex(/^\d{6}$/, '6桁の認証コードを入力してください'),
});

type SendOtpSchema = z.output<typeof emailOtpFormSchema>;
type VerifyOtpSchema = z.output<typeof emailOtpVerifySchema>;

const tabs = [
  { label: 'PitaMaiアカウント', value: 'oauth' },
  { label: 'メール認証', value: 'email' },
];

async function onSendOtp(event: FormSubmitEvent<SendOtpSchema>) {
  if (!turnstileToken.value) {
    toast.add({
      title: '確認が必要です',
      description: '「ロボットではありません」の認証を完了してください。',
      color: 'warning',
    });
    return;
  }

  loading.value = true;
  try {
    const { error } = await authClient.emailOtp.sendVerificationOtp({
      email: event.data.email,
      type: 'sign-in',
      fetchOptions: {
        headers: {
          'x-captcha-response': turnstileToken.value,
        },
      },
    });

    if (error) {
      console.error('Email OTP send error:', error);
      let errorMessage = 'エラーが発生しました。もう一度お試しください。';
      if (error.message) {
        errorMessage = error.message;
      } else if (error.status === 400) {
        errorMessage = 'メールアドレスの形式が正しくありません。';
      } else if (error.status === 500) {
        errorMessage =
          'サーバーエラーが発生しました。しばらくしてからお試しください。';
      }
      toast.add({
        title: 'エラー',
        description: errorMessage,
        color: 'error',
      });
      resetTurnstileToken();
      return;
    }

    emailState.email = event.data.email;
    otpSent.value = true;
    resetTurnstileToken();
    toast.add({
      title: '送信完了',
      description: '認証コードを送信しました。メールを確認してください。',
      color: 'success',
    });
  } catch (err) {
    console.error('Unexpected error:', err);
    const errorMessage =
      err instanceof Error
        ? err.message
        : 'エラーが発生しました。もう一度お試しください。';
    toast.add({
      title: 'エラー',
      description: errorMessage,
      color: 'error',
    });
    resetTurnstileToken();
  } finally {
    loading.value = false;
  }
}

async function onVerifyOtp(event: FormSubmitEvent<VerifyOtpSchema>) {
  if (!turnstileToken.value) {
    toast.add({
      title: '確認が必要です',
      description: '「ロボットではありません」の認証を完了してください。',
      color: 'warning',
    });
    return;
  }

  loading.value = true;
  try {
    const { error } = await authClient.signIn.emailOtp({
      email: emailState.email,
      otp: event.data.otp,
      callbackURL: '/apps/dashboard',
      errorCallbackURL: '/error',
      fetchOptions: {
        headers: {
          'x-captcha-response': turnstileToken.value,
        },
      },
    });

    if (error) {
      toast.add({
        title: 'エラー',
        description: error.message ?? '認証に失敗しました。コードを確認してください。',
        color: 'error',
      });
      resetTurnstileToken();
      return;
    }

    toast.add({
      title: 'ログイン成功',
      description: 'ダッシュボードへ移動します。',
      color: 'success',
    });
  } catch (err) {
    const errorMessage =
      err instanceof Error
        ? err.message
        : 'エラーが発生しました。もう一度お試しください。';
    toast.add({
      title: 'エラー',
      description: errorMessage,
      color: 'error',
    });
    resetTurnstileToken();
  } finally {
    loading.value = false;
  }
}

async function handleOAuthSignIn(provider: string) {
  oauthLoading.value = true;
  try {
    const { data, error } = await authClient.signIn.oauth2({
      providerId: provider,
      callbackURL: '/apps/dashboard',
      errorCallbackURL: '/error',
      scopes: ['openid', 'profile', 'email'],
    });

    if (error) {
      toast.add({
        title: 'エラー',
        description: error.message || 'PitaMaiアカウントでのサインインに失敗しました',
        color: 'error',
      });
      return;
    }

    toast.add({
      title: 'ログイン成功',
      description: 'ダッシュボードへ移動します。',
      color: 'success',
    });
  } catch (err) {
    const errorMessage =
      err instanceof Error
        ? err.message
        : 'エラーが発生しました。もう一度お試しください。';
    toast.add({
      title: 'エラー',
      description: errorMessage,
      color: 'error',
    });
  } finally {
    oauthLoading.value = false;
  }
}
</script>

<template>
  <div>
    <div v-if="session.data" class="flex items-center justify-center p-4">
      <UPageCard class="w-full max-w-md">
        <div class="flex flex-col items-center space-y-4 py-8">
          <UIcon name="i-lucide-check-circle" class="h-16 w-16 text-success" />
          <h2 class="text-xl font-semibold">ログイン済みです</h2>
          <p class="text-center">
            ようこそ、{{ session.data.user.name }}さん
          </p>
          <UButton to="/apps/dashboard" color="primary">
            ダッシュボードへ移動
          </UButton>
        </div>
      </UPageCard>
    </div>
    <div v-else class="flex items-center justify-center p-4">
      <UPageCard class="w-full max-w-md">
        <div class="space-y-6">
          <div>
            <h2 class="text-xl font-semibold">ログイン</h2>
            <p class="my-4 text-sm">
              ログイン方法を選択してください。
            </p>
            <UAlert description="PitaMaiアカウントでログインすることを推奨します。" class="text-sm" color="success" />
          </div>

          <!-- ログイン方法の選択タブ -->
          <UTabs :items="tabs" v-model="loginMethod" />

          <!-- メール認証セクション -->
          <div v-show="loginMethod === 'email'" class="space-y-4">
            <UAlert description="こちらのログイン方式は推奨されません。PitaMaiアカウントでのログインを推奨します。" color="error">
            </UAlert>
            <UForm v-if="!otpSent" :schema="emailOtpFormSchema" :state="emailState" class="space-y-4"
              @submit="onSendOtp">
              <UFormField label="メールアドレス" name="email" required>
                <UInput v-model="emailState.email" type="email" placeholder="user@email.com" autocomplete="email" />
              </UFormField>
              <UButton type="submit" :loading="loading" block>
                認証コードを送信
              </UButton>
            </UForm>

            <UForm v-else :schema="emailOtpVerifySchema" :state="otpState" class="space-y-4" @submit="onVerifyOtp">
              <p class="text-sm">
                {{ emailState.email }} に送信された6桁コードを入力してください。
              </p>
              <UFormField label="認証コード" name="otp" required>
                <UInput v-model="otpState.otp" placeholder="123456" maxlength="6" />
              </UFormField>
              <div class="flex gap-2">
                <UButton type="submit" :loading="loading" class="flex-1">ログイン</UButton>
                <UButton type="button" variant="outline" :disabled="loading" @click="otpSent = false" class="flex-1">
                  変更
                </UButton>
              </div>
            </UForm>

            <div v-if="config.public.TURNSTILE_SITE_KEY" id="login-turnstile" class="mt-4 flex justify-center" />
          </div>

          <!-- OAuth認証セクション -->
          <div v-show="loginMethod === 'oauth'" class="space-y-4">
            <div>
              <UButton variant="outline" :loading="oauthLoading" class="w-full flex items-center justify-center"
                @click="handleOAuthSignIn('custom')">
                <img src="/favicon.png" alt="PitaMaiロゴ画像" class="mr-2 h-8 w-8" />
                PitaMaiアカウント
              </UButton>
            </div>
          </div>
        </div>
      </UPageCard>
    </div>
  </div>
</template>
